# SaaS (Software as a Service)

SaaS is a **cloud delivery model** in which a software vendor hosts, secures, maintains, and upgrades an application
and exposes it to end-users over the internet — typically through a browser, on a **subscription** basis.
Instead of buying licenses and running the software on-premise, customers rent access to a managed service.

Interview questions usually drill into the **multi-tenancy** model, tenant isolation, billing, extensibility, and the
metrics that distinguish SaaS economics from on-premise software.

---

## Cloud Delivery Models — IaaS / PaaS / SaaS

| Model       | What the provider manages                 | What you manage                     | Examples                              |
|-------------|-------------------------------------------|-------------------------------------|---------------------------------------|
| **On-prem** | Nothing                                   | Everything (network, OS, app, data) | Self-hosted                           |
| **IaaS**    | Network, storage, servers, virtualization | OS, runtime, middleware, app, data  | EC2, GCE, Azure VM, DigitalOcean      |
| **PaaS**    | + OS, runtime, middleware                 | App, data                           | App Engine, Elastic Beanstalk, Heroku |
| **SaaS**    | + App                                     | Data (and configuration)            | Gmail, Salesforce, Slack, Notion      |

As you move down the list, the **vendor absorbs more operational responsibility** and the customer
gains speed of adoption but loses control over the underlying platform.

---

## Core SaaS Characteristics

1. **Centralized hosting** — a single (set of) service(s) serves many customers from the same deployment.
2. **Subscription / metered billing** — pay-as-you-go or per-seat pricing, recurring revenue.
3. **Multi-tenancy** — logical isolation of customer data inside a shared stack.
4. **Self-service onboarding** — sign-up, provisioning, and configuration without vendor involvement.
5. **Elastic scaling** — capacity grows with demand; no per-customer capacity planning.
6. **Continuous delivery** — one version of the software, frequent, transparent upgrades.
7. **High availability & DR** — vendor owns uptime SLAs, backups, multi-AZ failover.
8. **Configurable, not customizable** — extensibility via **settings, APIs, webhooks, plugins**, not forks.
9. **Security & compliance abstraction** — vendor covers SOC 2 / ISO 27001 / GDPR baseline; tenant owns its data.
10. **Observability & telemetry** — per-tenant usage, audit logs, and analytics are first-class products.

---

## Multi-Tenancy Models

The single most important architectural decision in a SaaS product. There is no "best" model — only trade-offs.

| Model                              | DB                                          | App tier           | Isolation     | Cost     | Noisy-neighbor risk | Best for                                              |
|------------------------------------|---------------------------------------------|--------------------|---------------|----------|---------------------|-------------------------------------------------------|
| **Silo**                           | 1 DB / tenant                               | 1 stack / tenant   | Very high     | High     | None                | Regulated / large enterprises                         |
| **Bridge (shared app, tenant DB)** | 1 DB / tenant                               | Shared             | High          | Medium   | Low                 | Mid-market, data-residency needs                      |
| **Pool** (shared schema)           | 1 shared DB — `tenant_id` column everywhere | Shared             | Low (logical) | Low      | High                | Consumer SaaS, free-tier productivity                 |
| **Hybrid**                         | Pool by default, silo on upgrade            | Shared + dedicated | Tiered        | Variable | Tiered              | Freemium → enterprise upgrade path (most modern SaaS) |

### Implementation: row-level isolation (Pool model)

Every query MUST filter by `tenant_id`. Forgetting it leaks data across tenants — the most cited SaaS bug.

```sql
-- ❌ dangerous — no tenant filter
SELECT id, title
FROM projects
WHERE status = 'ACTIVE';

-- ✅ correct
SELECT id, title
FROM projects
WHERE tenant_id = :tenantId
  AND status = 'ACTIVE';
```

PostgreSQL goes one step further with **Row-Level Security (RLS)** — the database enforces the filter, not the ORM:

```sql
CREATE
POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- App sets it once per request:
SET LOCAL app.tenant_id = '7e9f1c3a-...';
-- Now SELECT * FROM projects; cannot escape that tenant.
```

### Spring: enforce `tenant_id` for every request

```java

@Component
public class TenantInterceptor implements HandlerInterceptor {

    private final TenantContext ctx;

    public TenantInterceptor(TenantContext ctx) {
        this.ctx = ctx;
    }

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object h) {
        String tenantId = req.getHeader("X-Tenant-Id");
        if (tenantId == null) {
            resp.setStatus(401);
            return false;
        }
        ctx.setTenantId(UUID.fromString(tenantId));
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp,
                                Object h, Exception ex) {
        ctx.clear();   // critical — ThreadLocal leaks across pooled requests
    }
}
```

### Hibernate filter — apply tenant predicate globally

```java

@Entity
@FilterDef(name = "tenant", parameters = @ParamDef(name = "tenantId", type = UUID.class))
@Filter(name = "tenant", condition = "tenant_id = :tenantId")
public class Project { /* fields */
}
```

```java

@Component
public class TenantHibernateFilter {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void enable(UUID tenantId) {
        em.unwrap(Session.class)
                .enableFilter("tenant")
                .setParameter("tenantId", tenantId);
    }
}
```

---

## Identity, AuthN, AuthZ

| Layer              | Typical choice                                                            |
|--------------------|---------------------------------------------------------------------------|
| Authentication     | OIDC via Auth0 / Cognito / Auth0 / Keycloak                               |
| Authorization      | RBAC + ABAC; rows scoped by `tenant_id`                                   |
| Service-to-service | mTLS (Istio) or signed JWTs                                               |
| API access         | OAuth2 client-credentials; personal access tokens                         |
| Audit              | Every mutating action → `audit_log(tenant_id, actor, action, target, ts)` |

---

## Billing & Metering

SaaS revenue depends on a **metering pipeline** that counts billable events accurately and at scale.

```
  App event ──▶ Kafka "usage.events"
                  │
                  ├──▶ Stream processor (counts per tenant / per feature)
                  │         │
                  │         ▼
                  │   usage_ledger (append-only, immutable)
                  │         │
                  │         ▼
                  │   invoicing job (Stripe Billing / Adyen)
                  │
                  └──▶ Real-time dashboard (tenant usage, quota enforcement)
```

Two essential rules:

- **Immutability** — the ledger is append-only; corrections go in as new lines (`type = "CREDIT"` / `"REVERSAL"`).
- **Idempotency keys** — every usage event carries `event_id`; reprocessing cannot double-charge.

```java
public record UsageEvent(UUID eventId, UUID tenantId, String metric, long qty, Instant at) {
}

@PostMapping("/usage")
public ResponseEntity<Void> record(@RequestBody UsageEvent e) {
    if (ledger.exists(e.eventId()))        // idempotent
        return ResponseEntity.ok().build();
    ledger.append(e);
    publisher.publish(e);
    return ResponseEntity.accepted().build();
}
```

Common pricing models:

- **Per-seat** (Slack, Figma): `monthly = seats × unit_price`.
- **Tiered** (GitHub, Stripe-upgrade): flat fee per tier + overage.
- **Metered** (Twilio, Snowflake): `Σ(qty_i × unit_price_i)`.
- **Hybrid**: base subscription + consumption over a quota.

---

## SaaS Metrics

| Metric              | Definition                                                | Healthy SaaS range      |
|---------------------|-----------------------------------------------------------|-------------------------|
| MRR / ARR           | Monthly / annual recurring revenue                        | —                       |
| CAC                 | Customer acquisition cost (sales + marketing)             | —                       |
| LTV                 | Gross margin × (avg customer lifespan)                    | LTV / CAC ≥ 3           |
| Churn (logo)        | % of customers lost per period                            | < 5 % / yr (B2B SMB)    |
| NRR                 | (start MRR + expansion − churn − contraction) / start MRR | > 100 % (net retention) |
| Gross margin        | (Revenue − COGS) / Revenue                                | SaaS: 70–85 %           |
| Time-to-value (TTV) | Sign-up → first "aha" moment                              | Days, not weeks         |

Net Revenue Retention > 100% means existing customers grow faster than churn shrinks the book — the defining
economic engine of a healthy SaaS.

---

## Code Generation in SaaS

A surprising amount of a SaaS platform's surface area is **generated**, not hand-written. Three recurring patterns:

### 1. Schema-driven CRUD generation (admin/back-office)

Tenants configure custom **entities** (think Salesforce "Objects" or Airtable "Tables"). The SaaS engine generates
DTOs, REST controllers, persistence mappings, and even React forms from a runtime schema.

```java
// Tenant-defined schema — stored as JSON metadata
public record EntitySchema(String name, List<Field> fields) {
}

public record Field(String name, FieldType type, boolean required) {
}

public enum FieldType {STRING, NUMBER, DATE, REF}
```

```java

@Component
public class DynamicRecordRepository {

    private final JdbcTemplate jdbc;

    /** Generates SQL + bind values from a schema and a tenant-provided payload. */
    public void insert(EntitySchema schema, UUID tenantId, Map<String, Object> payload) {
        var cols = new ArrayList<String>(List.of("tenant_id"));
        var vals = new ArrayList<Object>(List.of(tenantId));
        for (Field f : schema.fields()) {
            if (!payload.containsKey(f.name()) && f.required())
                throw new BadRequestException("Missing " + f.name());
            cols.add(f.name());
            vals.add(payload.get(f.name()));
        }

        String sql = "INSERT INTO %s (%s) VALUES (%s)".formatted(
                snake(schema.name()),
                String.join(",", cols),
                String.join(",", Collections.nCopies(vals.size(), "?")));

        jdbc.update(sql, vals.toArray());
    }
}
```

For higher expressive power, **EMF / Spring Data JDBC generators** or **JOOQ schema-based fetching** can be used; the
common point is: the platform turns tenant-supplied *metadata* into runtime behavior.

### 2. SDK & client generation from OpenAPI

Tenants automate against your API with generated typed clients.

```yaml
# openapi.yaml (excerpt)
components:
  schemas:
    Project:
      type: object
      required: [ id, tenantId, title ]
      properties:
        id: { type: string, format: uuid }
        tenantId: { type: string, format: uuid }
        title: { type: string, maxLength: 120 }
```

```
openapi-generator-cli generate \
    -i openapi.yaml \
    -g typescript-axios \
    -o clients/ts
```

The TypeScript client and Java SDK downloaded by tenants are **the same generator run against the contract**, so
breaking changes are caught at the contract level — not in tenant integration code.

### 3. Scaffolding tenant resources (IaC generation)

Onboarding a new tenant may provision a database namespace, an OAuth2 client, a dedicated queue, and a feature-flag
set. We **generate** Terraform rather than hand-creating the environment:

```hcl
# tenant.tf — generated per onboarding
module "tenant" {
  source            = "./modules/tenant"
  tenant_id         = "7e9f1c3a-..."
  tenant_slug       = "acme"
  dedicated_db      = var.plan == "ENTERPRISE"
  region            = data.aws_caller_identity.current.region
  feature_flags     = local.flags[var.plan]
}
```

```python
# onboarding.py (excerpt)
import subprocess, json, pathlib

def generate_tf(tenant: dict) -> str:
    tf = pathlib.Path("modules/tenant/main.tf").read_text()
    return tf.format(**tenant)

def onboard(tenant):
    pathlib.Path(f"tenants/{tenant['slug']}.tf").write_text(generate_tf(tenant))
    subprocess.run(["terraform", "-chdir=tenants", "apply", "-auto-approve"], check=True)
```

A single onboarding event produces reproducible infrastructure — auditable, version-controlled, and revocable with
`terraform destroy`.

---

## Tenancy at Scale — Hybrid Model

Most modern SaaS start free-tier customers on the **Pool** model and upgrade paying enterprise accounts to a **Bridge**
or **Silo** stack on the same code base. The flow:

```
  Sign-up ──▶ Pool (shared DB, shared app, stripe-billed)
                │
                │ Enterprise upgrade
                ▼
   Migration job ──▶ dedicated schema/DB
                │
                ▼
   Bridge / Silo (same code path, different TenantRouter binding)
```

The application code never branches on plan — a `TenantRouter` decides where each request lands:

```java
public interface TenantRouter {
    DataSource forTenant(UUID tenantId);
}

@Component
public class DefaultTenantRouter implements TenantRouter {

    private final DataSource pool;          // shared
    private final Map<UUID, DataSource> dedicated;  // per big tenant

    public DataSource forTenant(UUID tenantId) {
        return dedicated.getOrDefault(tenantId, pool);
    }
}
```

This keeps **operational extremes** (one DB per customer vs. one DB for all) behind a single policy seam — the kind
of design interviewers love to probe.

---

## Common Interview Questions

- What is SaaS? Compare it with IaaS and PaaS, and explain why the choice matters for a startup vs. a bank.
- Describe multi-tenancy models. Which would you choose for a healthcare-focused SaaS and why?
- How do you prevent a single query from leaking data across tenants in a shared-schema app?
- Design the metering pipeline for a usage-based SaaS (e.g. Twilio). How do you keep it idempotent?
- A new enterprise customer insists on its own database. How do you stay on the same code base?
- What is Net Revenue Retention, and why is it more important than acquisition for mature SaaS?
- How would you implement tenant-scoped rate limiting and quotas?
- Walk through the migration of a tenant from shared to dedicated infrastructure **without downtime**.
- How does your OpenAPI-driven SDK generation prevent accidental contract breakage when shipping a new API version?

---

## Related

- [Docker Core Concepts](../Docker/docker-core-concepts.md) — containers underpin elastic SaaS deployments.
- [Kubernetes Core Concepts](../Kubernetes/kubernetes-core-concepts.md) — multi-tenant scheduling, namespaces, quotas.
- [REST Principles](../../System-Design/API-Design/rest-principles.md) — versioning, contracts, idempotency keys.
- [Microservices Architecture](../../Java/spring/cloud/microservices-architecture.md) — service boundaries in a
  tenant-aware stack.
- [Spring Cloud Ecosystem](../../Java/spring/cloud/spring-cloud-ecosystem.md) — config, discovery, gateway per tenant.

---

## Resources

- *Architecting for Multi-Tenant SaaS* — AWS Whitepaper (SaaS factory program).
- *Building Multi-Tenant SaaS Applications* — video series, AWS re:Invent.
- *SaaS Metrics That Matter* — David Skok, *For Entrepreneurs*.
- *The OpenAPI Specification* — https://spec.openapis.org/oas/latest.html
- *Multi-Tenant Data Architectures* — Microsoft Patterns & Practices.