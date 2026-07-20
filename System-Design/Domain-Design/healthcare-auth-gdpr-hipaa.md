# Healthcare Authentication & Authorization — GDPR / HIPAA

Healthcare is one of the most regulated domains for software engineering. Patient data is **both** the most
sensitive category most systems handle **and** the most shared — across hospitals, labs, insurers, pharmacies,
researchers, and the patient themselves. Auth in this domain is therefore **not** a single feature but an
**architecture**: identity, consent, scope, audit, and encryption are inseparable from the regulatory landscape.

A frequent senior/staff interview topic at health-tech companies (Epic, Cerner, Teladoc, Doximity, Babylon,
Tempus, insurance payers) and at any vendor selling B2B SaaS into hospitals. Expect to discuss SMART on FHIR,
break-glass, minimum-necessary, audit trails, consent capture, and the differences between HIPAA and GDPR.

---

## Quick Reference — Regulations & Their Auth Implications

| Regulation            | Region            | What it protects                                | Key auth-relevant rules                                       |
|-----------------------|-------------------|-------------------------------------------------|---------------------------------------------------------------|
| **HIPAA**             | USA               | PHI (Protected Health Information)              | Access controls, audit logs, minimum necessary, encryption    |
| **HITECH**            | USA               | Extends HIPAA                                   | Breach notification, meaningful-use EHR incentives            |
| **GDPR**              | EU/EEA            | Personal data + **special categories** (health) | Lawful basis, **explicit consent**, data subject rights, DPO  |
| **EHDS** (EU Health Data Space) | EU       | Cross-border health data exchange       | Strengthens GDPR for health; secondary-use opt-in             |
| **PIPEDA**            | Canada            | Personal information incl. health               | Consent, safeguards                                           |
| **DPA 2018 / UK GDPR**| UK                | Health data                                     | Mirrors GDPR with UK-specific provisions                      |
| **HITRUST CSF**       | USA (de-facto)    | Certifiable control framework                   | Maps HIPAA + ISO 27001 + NIST → auditable implementation      |
| **21 CFR Part 11**    | USA               | Electronic records in clinical trials           | Signed e-records, audit trails                                |
| **CCPA/CPRA**         | California, USA   | Personal info (incl. medical)                   | Disclosure rights; complements HIPAA where not preempted      |

> **Rule of thumb:** HIPAA says **how** to protect PHI (safeguards, audits, minimum-necessary). GDPR says **when**
> you may process personal data **at all** (lawful basis — for health data, usually explicit consent). Most
> healthcare apps must satisfy **both**.

---

## 1. The Domain Vocabulary

| Term                  | Meaning                                                                    |
|-----------------------|----------------------------------------------------------------------------|
| **PHI** (HIPAA)       | Protected Health Information — any health data + identifier (name, MRN, IP)|
| **ePHI**              | PHI in electronic form                                                     |
| **PII**               | Personally Identifiable Information — broader than PHI                     |
| **Special category data** (GDPR Art. 9) | Health, genetics, biometrics, etc. — **explicit consent required** |
| **MRN**               | Medical Record Number — patient identifier inside one institution          |
| **EHR / EMR**         | Electronic Health/Medical Record                                           |
| **FHIR**              | **F**ast **H**ealthcare **I**nteroperability **R**esources — HL7 REST API  |
| **SMART on FHIR**     | OAuth2 profile for healthcare apps launching from an EHR                   |
| **Covered entity**    | Provider / payer / clearinghouse that must comply with HIPAA               |
| **Business Associate (BA)** | Vendor handling PHI on behalf of a covered entity — needs a BAA    |
| **BAA**               | Business Associate Agreement — contractually extends HIPAA to the vendor   |
| **Minimum necessary** | HIPAA requirement: access only what's needed for the task                  |
| **Break-glass**       | Emergency access to records outside normal privileges, heavily audited     |

---

## 2. The Two Frameworks Compared

### 2.1 HIPAA — "How to protect"

HIPAA Security Rule (45 CFR §164) mandates **administrative, physical, technical safeguards**. Auth-relevant
technical safeguards:

- **Access control** (§164.312(a)) — unique user identification, emergency access (break-glass), automatic logoff,
  encryption/decryption.
- **Audit controls** (§164.312(b)) — hardware/software procedures that record and examine activity in systems
  containing ePHI.
- **Integrity** (§164.312(c)) — protect ePHI from improper alteration.
- **Person or entity authentication** (§164.312(d)) — verify identity.
- **Transmission security** (§164.312(e)) — integrity + encryption in transit.

Supporting rules:
- **Minimum necessary** (§164.502(b), §164.514(d)) — uses/disclosures limited to the minimum needed.
- **Sanction policy** — penalties for workforce members who violate.
- **Risk analysis** (§164.308(a)(1)) — ongoing, documented risk assessments.

HIPAA does **not** prescribe exact technologies (no "use OAuth2"). It requires *reasonable and appropriate*
controls. Auditors look for the **paper trail**: risk assessments, policies, access logs, training.

### 2.2 GDPR — "When you may process at all"

GDPR Art. 9 lists health data as a **special category** — processing is **prohibited** unless one of ten
exemptions applies. The most relevant for healthcare apps:

- **(a) Explicit consent** — opt-in, withdrawable, granular.
- **(h)** — medical diagnosis, treatment, or management of health services by **professionals** under obligation
  of professional secrecy.
- **(i)** — public health threat.
- **(j)** — scientific research with appropriate safeguards.

GDPR also grants **data subject rights**:

- **Access** (Art. 15) — "show me everything you have on me."
- **Rectification** (Art. 16).
- **Erasure** (Art. 17) — "right to be forgotten" (with medical-record retention exceptions).
- **Restriction** (Art. 18).
- **Portability** (Art. 20).
- **Object** (Art. 21).

Auth implications:

- The **auth system must carry consent state** — what the user agreed to, when, scope by scope.
- **Audit logs are themselves personal data** — subject to the same rights.
- **Erasure is hard**: medical records often have **legal retention minimums** (e.g., 7–10 years) that override
  the right to be forgotten.

### 2.3 Where they diverge

| Question                       | HIPAA                       | GDPR                              |
|--------------------------------|-----------------------------|-----------------------------------|
| Lawful basis to process?       | Treatment/payment/operations| Explicit consent or Art. 9(h)     |
| Consent revocable?             | Not really (treatment basis)| Yes, immediately                  |
| Cross-border transfer?         | Permitted (with BAAs)       | Restricted — SCCs, adequacy       |
| Right to be forgotten?         | No (retention mandated)     | Yes, with exceptions              |
| Audit-log retention            | 6 years minimum             | No fixed minimum                  |
| Encryption mandated?           | "Addressable"               | "Appropriate technical measures"  |
| Notification of breach         | ≤ 60 days                   | **≤ 72 hours** to DPA             |

---

## 3. Identity & Authentication (AuthN)

### 3.1 Identity tiers

Healthcare auth involves **multiple actor classes** with very different requirements:

| Actor                  | Typical identity source                          | MFA?  |
|------------------------|--------------------------------------------------|-------|
| Physician / clinician  | Hospital IdP (Active Directory, Cerner, Epic)    | Yes   |
| Nurse / staff          | Same IdP, different role group                   | Yes   |
| Patient (portal/app)   | Patient IdP (email, phone, national e-ID)        | Yes (step-up) |
| Researcher             | Federated (inCommon, eduGAIN, ORCID)             | Yes   |
| Payer / insurer system | Service account / mTLS                           | n/a   |
| Device (IoT, imaging)  | X.509 cert / device attestation                  | n/a   |

### 3.2 Standards

- **OAuth 2.0** (RFC 6749) — delegated authorization framework. Universal baseline.
- **OpenID Connect (OIDC)** — identity layer on top of OAuth2; provides `id_token` (JWT) with user claims.
- **SAML 2.0** — still dominant inside hospitals (legacy IdP integration, ADFS, Shibboleth).
- **FIDO2 / WebAuthn** — phishing-resistant MFA; recommended for clinicians.
- **mTLS + JWT** — service-to-service auth in microservices (with SPIFFE/SPIRE or internal PKI).

### 3.3 SMART on FHIR

SMART (Substitutable Medical Applications, Reusable Technologies) is the **OAuth2 profile for healthcare apps**.
It defines:

- How a third-party app launches from an EHR (EHR-launch vs. standalone).
- Scopes that map to FHIR resources: `patient/Observation.read`, `user/AllergyIntolerance.write`,
  `launch/patient`, `openid`, `fhirUser`.
- The **patient context** (`launch/patient`, `launch/encounter`) — the EHR tells the app *which patient*.
- PKCE for public clients (mandatory for browser/mobile).
- Asymmetric JWT client authentication (`client_credentials` with `private_key_jwt`).

Typical launch flow:

```
   ┌────────────┐  1. User in EHR clicks "Launch App"
   │   EHR      │ ─────────────────────────────────────┐
   │ (patient)  │                                      ▼
   └────────────┘                          ┌──────────────────────┐
   ┌────────────┐  2. Browser redirect     │   SMART App           │
   │  Browser   │ ◀────────────────────────│   (third-party)       │
   │            │  3. PKCE auth code flow  │                       │
   │            │ ────────────────────────▶│  EHR /authorize       │
   │            │  4. Auth code            │                       │
   │            │ ◀────────────────────────│                       │
   │            │  5. Exchange for tokens  │                       │
   │            │                          │  EHR /token (JWT)     │
   └────────────┘                          │                       │
   ┌────────────┐                          │   6. FHIR API calls   │
   │  EHR FHIR  │ ◀────────────────────────│   with Bearer access  │
   │  /Patient  │                          │   token + scope       │
   │  /Obs ...  │ ────────────────────────▶│                       │
   └────────────┘                          └──────────────────────┘
```

The `access_token` is a **JWT** carrying `scope`, `patient`, `encounter` claims. EHRs enforce scopes per resource.

### 3.4 MFA and step-up auth

- HIPAA doesn't *mandate* MFA but auditors expect it; **HITECH meaningful-use** programs effectively require it.
- Use **step-up** for high-risk actions (writing a prescription for controlled substances, viewing another
  employee's record, exporting data).
- DEA EPCS (Electronic Prescribing of Controlled Substances) requires **two-factor** with **hard tokens** or
  biometrics — soft OTPs are not enough.

### 3.5 Session & automatic logoff

HIPAA §164.312(a)(2)(iii) requires **automatic logoff**. Implement:

- Short access-token TTL (5–15 min) + refresh token rotation.
- Idle-session timeout in the UI (10–15 min).
- SSO logout propagation (OIDC RP-Initiated Logout, SAML Single Logout).
- Token revocation on consent withdrawal.

---

## 4. Authorization (AuthZ)

Healthcare authorization is **multi-dimensional** — same user may have different access depending on context.

### 4.1 Access models

| Model            | Use case                                              |
|------------------|-------------------------------------------------------|
| **RBAC**         | Base layer — role defines capability ("Physician")    |
| **ABAC**         | Context — `patient.assignedPhysician == user`         |
| **Relationship-based** | "I am the patient's treating clinician"        |
| **TBAC** (Time)  | On-call windows, shift boundaries                     |
| **PBAC** (Policy)| Centralised policies (XACML, OPA, Cedar)              |
| **LBAC** (Lattice)| Classified labels (HIV status, mental health, etc.)  |

In practice, real systems combine RBAC + ABAC + relationship checks.

### 4.2 Minimum-necessary enforcement

Not just policy — enforce **per request**:

- A billing clerk may view demographic + financial data but **not** clinical notes.
- A clinician may view their **own patients** but not the entire clinic.
- Front-desk staff may view the schedule but **not** lab results.

Implement via:

- **Row-level security** at the data layer (PostgreSQL RLS, row filters in ORM).
- **Field-level redaction** (mask specific JSON fields in API responses).
- **Purpose-of-use tags** on the request, enforced in policy.

### 4.3 Break-glass

Emergency access outside normal privileges — e.g., an ER physician treating an unconscious patient who isn't on
their patient list.

Design pattern:

```
   1. User clicks "Emergency access".
   2. App records: who, when, why (free-text reason), whose record.
   3. Access is granted for a bounded time (e.g., 1 hour).
   4. Event goes to a SEPARATE audit log that triggers an alert.
   5. Compliance officer reviews every break-glass event within 24-72h.
```

Break-glass must be:

- **Possible** — no barrier in a true emergency.
- **Auditable** — every use is reviewed.
- **Sanction-able** — repeated unjustified use has consequences.

### 4.4 Consent-aware authz

GDPR elevates consent to a **first-class authz input**. A patient may consent to:
- General practitioner seeing everything.
- Researcher seeing de-identified records only.
- Insurer seeing claims only.

Consent has its own lifecycle:

```
   ┌──────────────┐  capture   ┌──────────────┐  enforce   ┌──────────────┐
   │  Consent UI  │ ─────────▶ │ Consent      │ ─────────▶ │  Authz PDP   │
   │  (granular)  │            │  Registry    │            │  (policy)    │
   └──────────────┘            └──────────────┘            └──────────────┘
                                       │
                                       │ versioned, immutable history
                                       ▼
                                 Consent Audit Log
```

Key design rules:
- **Versioned, immutable** — store every change with timestamp and reason.
- **Per-purpose** — separate consent for treatment, payment, operations, research, marketing.
- **Revocable at any time** — propagates to Authz in seconds (cache invalidation!).
- **Cross-references the auth token** — claims or scopes carry consent hash/version.

### 4.5 Patient-as-data-subject access

GDPR Art. 15 + HIPAA Right of Access (45 CFR §164.524) — patients can request their records. Patterns:

- Self-service portal with **read** access to their own FHIR resources.
- Formal request workflow for redacted/anonymized exports.
- Rate-limit and audit every export; data exfiltration concerns are real.

---

## 5. Audit Logging

HIPAA §164.312(b) and GDPR Art. 30 + Art. 32 make audit logging **mandatory**, not optional.

### 5.1 What to log

For every access to PHI:

```
   WHO       user identity + role
   WHAT      action (READ, WRITE, EXPORT, DELETE, BREAK_GLASS)
   WHICH     patient + resource type + resource ID + fields
   WHEN      timestamp (UTC, immutable)
   WHERE     source IP, device, geo (if known)
   WHY       purpose of use (treatment, payment, operations, research)
   HOW       auth method, session id, correlation id / trace id
   RESULT    success / denial / reason
```

### 5.2 Tamper-proofing

Audit logs are evidence — both for regulators and in lawsuits. They must resist modification:

- **Append-only** storage (Postgres with `REVOKE UPDATE`, DynamoDB with conditional writes, AWS QLDB, immudb).
- **Hash-chained** — each entry includes the hash of the previous (Merkle/tamper-evident log).
- **WORM** — write-once storage (S3 Object Lock, Azure Immutable Blob).
- **Cross-system correlation** — store the same correlation ID in the auth token, API log, and DB log.

### 5.3 Retention

- HIPAA: **6 years** from creation or last effective date (longer if state law requires; medical-record retention
  often 7–10 years post-discharge, longer for minors).
- GDPR: no fixed minimum, but you must justify retention; **no maximum either** if legally required.
- Patient audit-disclosure logs: HIPAA gives patients the right to an **accounting of disclosures** for the prior
  6 years (extended to all TPO disclosures under HITECH, with implementation pending).

### 5.4 What logs are themselves

GDPR: audit logs containing identifiers are **personal data** and subject to data subject rights. Practical fix:

- Pseudonymize user identifiers in logs (store user_id; map to name in a separate table).
- Use **purpose limitation** — logs are for security/compliance, not marketing.

---

## 6. Encryption & Token Strategy

### 6.1 The three layers

| Layer          | HIPAA expectation        | Modern practice                                       |
|----------------|--------------------------|-------------------------------------------------------|
| In transit     | TLS 1.2+ "addressable"   | TLS 1.3 everywhere; mTLS between services             |
| At rest        | "addressable"            | AES-256 (KMS-managed keys) on all stores              |
| Field/column   | Implied by minimum-needed| Per-field encryption for highest-sensitivity fields   |

> "Addressable" in HIPAA means: implement, or document why an equivalent control is in place. In practice,
** encryption is universally expected** and the cheapest defense.

### 6.2 Field-level encryption

Encrypt entire FHIR resources, or just the most sensitive fields (mental health notes, HIV status, substance
abuse treatment records — which under **42 CFR Part 2** are even **more** restricted than HIPAA).

```java
// Pattern: encrypt before persist, decrypt on read, with attribute-based access
class PatientRepository {
    private final FieldEncryptor encryptor;   // AES-GCM, KMS-backed

    public void save(Patient p) {
        if (p.hasSensitiveNotes()) {
            p.setNotes(encryptor.encrypt(p.getNotes(), keyFor(p.getId())));
        }
        jpa.save(p);
    }
}
```

### 6.3 Tokens

- **Access tokens: short TTL (5–15 min), JWT** carrying claims: `sub`, `scope`, `patient`, `encounter`,
  `purpose_of_use`, `consent_version`, `session_id`.
- **Refresh tokens: longer, rotated on every use** (OAuth2 refresh-token rotation with reuse detection).
- **No PHI in the token body** — JWT is base64-encoded, not encrypted; only claims needed for authz.
- **Subject identifier pseudonymization** — internal `sub` ≠ MRN; mapping table is itself PHI.
- **Token revocation** — keep a deny-list; check on every request (cache 30s). Critical for consent withdrawal.

### 6.4 Key management

- **Never** hard-code keys.
- Use a KMS (AWS KMS, Azure Key Vault, GCP KMS, HashiCorp Vault).
- **Separate keys per tenant** in multi-tenant SaaS — so a leaked key compromises one tenant, not all.
- **Envelope encryption**: KMS encrypts data-encryption-keys (DEKs); DEKs encrypt data. Avoids KMS rate limits.

---

## 7. Cross-Organization Interoperability

Health data flows across organizations. Common patterns:

### 7.1 IHE profiles

- **IHE XUA** (Cross-Enterprise User Assertion) — SAML assertion propagating user identity across enterprises.
- **IHE BPPC** (Basic Patient Privacy Consents) — machine-enforceable consent assertions.
- **IHE ATNA** (Audit Trail and Node Authentication) — standardised audit messages (RFC 3881).

### 7.2 TEFCA & QHINs (US, 2023+ )

The Trusted Exchange Framework and Common Agreement creates **QHINs** (Qualified Health Information Networks) —
a federated trust layer for nationwide exchange. Identity, consent, and audit requirements are codified by the
Common Agreement.

### 7.3 European Health Data Space (EHDS)

Cross-border infrastructure for primary (care) and secondary (research) use. Builds on GDPR with **mandatory
interoperability** requirements and EHR exchange format standards.

### 7.4 Service-to-service trust

For B2B / payer-provider / lab-hospital integrations:

- **mTLS** with certificates issued by a recognised CA (or private PKI).
- **OAuth2 client_credentials** with `private_key_jwt` for sender assertion.
- **Signed FHIR payloads** (JWS) for non-repudiation.
- **Per-call audit** — both sides log every transaction.

---

## 8. Implementation Patterns

### 8.1 Reference architecture

```
   ┌───────────────────────────────────────────────────────────────────────────┐
   │                            AUTH FABRIC                                     │
   │                                                                            │
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
   │  Patient    │   │  Clinician  │   │  SMART App  │   │  Partner System │   │
   │   portal    │   │   EHR/EMR   │   │             │   │  (payer/lab)    │   │
   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └────────┬────────┘   │
          │                  │                 │                   │            │
          ▼                  ▼                 ▼                   ▼            │
   ┌─────────────────────────────────────────────────────────────────────────┐  │
   │                     IdP / Authorization Server                          │  │
   │   OIDC + OAuth2 + SMART on FHIR + SAML gateway + MFA + step-up         │  │
   └─────────────────────────────┬───────────────────────────────────────────┘  │
                                  │ JWT + scope + consent_version              │
                                  ▼                                             │
   ┌─────────────────────────────────────────────────────────────────────────┐  │
   │                       API Gateway / PEP                                  │  │
   │   Validate signature, expiry, scope, consent; rate-limit; audit entry   │  │
   └─────────────────────────────┬───────────────────────────────────────────┘  │
                                  │                                             │
              ┌───────────────────┼───────────────────┐                         │
              ▼                   ▼                   ▼                         │
   ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
   │ Policy PDP      │  │ Consent Registry │  │ Audit Log (WORM) │             │
   │ (OPA / Cedar /  │  │ versioned        │  │ hash-chained     │             │
   │  XACML)         │  │ immutable        │  │ correlation ID   │             │
   └─────────────────┘  └──────────────────┘  └──────────────────┘             │
              │                                                                │
              ▼                                                                │
   ┌─────────────────────────────────────────────────────────────────────────┐  │
   │   FHIR services / domain APIs (encrypted at rest + per-field)           │  │
   └─────────────────────────────────────────────────────────────────────────┘  │
   └───────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Policy decision point (PDP) example — OPA/Rego

```rego
package healthcare.authz

default allow := false

# A clinician can read a patient's record if:
#   - the token scope includes patient/*.read
#   - the patient is on their panel (relationship)
#   - consent for treatment is in force
allow if {
    input.token.scope[_] == "patient/*.read"
    input.token.role == "physician"
    relationship(input.user_id, input.patient_id)
    consent_active(input.patient_id, "treatment")
    input.action == "READ"
}

# Break-glass: always allow but always audit
allow if {
    input.break_glass == true
    input.reason != ""
}
```

The API gateway calls the PDP on every request; the PDP combines token, relationship, and consent.

### 8.3 Spring Security example (SMART on FHIR resource server)

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/fhir/**").hasAuthority("SCOPE_patient/*.read")
                .requestMatchers("/fhir/**").access(
                    new WebExpressionAuthorizationManager(
                        "@patientPolicy.allow(authentication, request)"))
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .csrf(CsrfConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .addFilterBefore(new AuditFilter(), AuthorizationFilter.class)
            .build();
    }
}

@Component("patientPolicy")
public class PatientPolicy {
    private final RelationshipService relationships;
    private final ConsentService consents;

    public boolean allow(Authentication auth, HttpServletRequest req) {
        var jwt   = (Jwt) auth.getPrincipal();
        var scope = jwt.getClaimAsString("scope");
        var patientId = extractPatient(req);
        var consentVersion = jwt.getClaimAsString("consent_version");
        return scope.contains("patient/*.read")
            && relationships.isTreating(auth.getName(), patientId)
            && consents.isActive(patientId, "treatment", consentVersion);
    }
}
```

### 8.4 Audit filter (capture on every request)

```java
public class AuditFilter extends OncePerRequestFilter {

    private final AuditPublisher audit;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        var start = Instant.now();
        try {
            chain.doFilter(req, res);
        } finally {
            var user = SecurityContextHolder.getContext().getAuthentication().getName();
            audit.publish(AuditEvent.builder()
                .who(user)
                .what(req.getMethod() + " " + req.getRequestURI())
                .result(res.getStatus())
                .when(start)
                .correlationId(MDC.get("traceId"))
                .build());
        }
    }
}
```

The publisher writes to the **append-only** store with hash-chaining; never to the OLTP database.

---

## 9. Common Patterns & Anti-Patterns

### ✅ Do

- **Build authz on top of a PDP** (OPA, Cedar) — keep policy **out of the code**.
- **Carry `purpose_of_use` and `consent_version` in the JWT** — auditable end-to-end.
- **Use SMART on FHIR scopes** for interop; map internal roles to them.
- **Encrypt per-field** for the most sensitive categories (HIV, mental health, substance use — 42 CFR Part 2).
- **Hash-chain audit logs** and write them to immutable storage.
- **Validate consent on every read**, not just on login.
- **Force step-up MFA** for high-risk actions (controlled-substance Rx, export, employee-record access).
- **Pseudonymize user identifiers in logs** — logs are themselves personal data under GDPR.
- **Use mTLS** between services; issue short-lived service identities (SPIFFE).
- **Sign BAA** with every vendor before any PHI touches them.

### ❌ Don't

- **Don't** put PHI in a JWT body — it's only base64-encoded.
- **Don't** rely on RBAC alone — relationship and consent are required.
- **Don't** keep refresh tokens indefinitely — rotate + reuse detection.
- **Don't** write audit logs to the OLTP database — corruption risk, SQL UPDATE risk.
- **Don't** let the patient portal session stay valid after consent withdrawal.
- **Don't** log full request/response bodies — they may contain PHI you didn't intend to log.
- **Don't** expose sequential patient IDs (MRN) in URLs — use opaque UUIDs.
- **Don't** ship the BAA "later" — it's a precondition for any PHI handling.

---

## 10. Common Interview Questions

1. **What's the difference between HIPAA and GDPR for healthcare data?**
   HIPAA governs **how** to safeguard PHI (security rule, minimum necessary, audit). GDPR governs **when** you may
   process health data at all (Art. 9 — explicit consent or treatment-by-professionals exemption) and grants data
   subject rights (access, erasure, portability). Health apps targeting both markets must satisfy both.

2. **What is SMART on FHIR?**
   An OAuth2 profile for healthcare apps. It defines scopes that map to FHIR resources (`patient/Observation.read`),
   patient/encounter launch context, PKCE for public clients, and asymmetric client auth. SMART lets a clinician
   launch a third-party app from inside an EHR with narrowly-scoped, time-limited access to that patient's
   resources.

3. **How does break-glass access work, and why is it audit-heavy?**
   Break-glass grants emergency access outside a user's normal privileges (e.g., ER doctor treating an unconscious
   non-panel patient). It must always be **possible** (no barrier in an emergency), but every use is logged and
   reviewed by compliance. The user provides a reason; the access is time-bounded; the event is flagged for review
   within 24–72 hours.

4. **How do you enforce minimum-necessary?**
   Three layers: (1) row-level security so users see only their own patients; (2) field-level redaction so
   non-treating roles can't see clinical notes; (3) purpose-of-use tags enforced by a policy decision point.

5. **How is consent captured and enforced end-to-end?**
   Consent is captured per-purpose (treatment, payment, research, marketing), versioned immutably. The current
   consent version is referenced in the JWT (`consent_version` claim). The PDP checks consent on **every** read.
   Revocation invalidates tokens within seconds (deny-list + cache invalidation).

6. **Where should PHI never appear?**
   - JWT body (it's only base64-encoded).
   - URLs (logged by proxies, browsers, Referer).
   - Plain logs — only correlation IDs and pseudonyms.
   - Error messages or stack traces.
   - Client-side storage (localStorage) without explicit encryption and justification.

7. **What is the difference between a covered entity and a business associate?**
   A covered entity (provider, payer, clearinghouse) is the primary HIPAA-regulated organisation. A business
   associate is any vendor that handles PHI on its behalf. The relationship requires a **BAA** — without one, no
   PHI may be shared.

8. **How do you tamper-proof audit logs?**
   Append-only storage (WORM, S3 Object Lock, QLDB); hash-chaining where each entry includes the hash of the
   previous; cross-system correlation IDs; periodic external attestation. The goal is non-repudiable evidence.

9. **What is 42 CFR Part 2 and why does it matter?**
   US federal regulation covering substance-use-disorder treatment records. **Stricter** than HIPAA — even the
   fact that someone is in treatment is restricted. Requires **explicit re-consent** for re-disclosure. Many EHRs
   segment these records entirely.

10. **How do you handle the GDPR right to erasure when medical records must be retained?**
    GDPR Art. 17 has an exception for "compliance with a legal obligation" — medical-record retention laws
    override erasure. The right response is: explain the retention requirement, restrict processing, and erase
    after the retention period ends. Pseudonymisation can help — strip the identifying mapping so the remaining
    clinical data is no longer personal data.

11. **How do you design auth for a multi-tenant EHR SaaS?**
    Separate KMS keys per tenant; tenant ID in the JWT; row-level security enforced on every query; per-tenant
    audit log streams; per-tenant consent registry; isolated token signing keys (or tenant-keyed JWT verification
    via JWKS endpoint selection).

12. **How do you secure service-to-service auth in a healthcare microservices system?**
    mTLS between services (SPIFFE for short-lived identities); OAuth2 client_credentials with `private_key_jwt`;
    signed FHIR payloads (JWS) for non-repudiation; per-call audit on both sides; no shared service accounts — one
    identity per service per tenant.

13. **How does the right of access (GDPR Art. 15) get implemented technically?**
    A patient-facing portal where they can browse their own FHIR resources (read scope on their own patient
    compartment); a formal request workflow for exports of full records (including audit log entries about them);
    rate-limited and heavily audited; pseudonymised identifiers in logs must be re-identified only for the data
    subject.

14. **What breach-notification timelines apply?**
    HIPAA: notify affected individuals ≤ **60 days**; HHS annually for < 500-individual breaches, ≤ 60 days for
    ≥ 500. GDPR: notify the supervisory authority ≤ **72 hours**; affected individuals "without undue delay" if
    high risk. Build runbooks to meet both.

15. **What is the role of a PDP like OPA in healthcare auth?**
    Externalise authorisation policy from the code so policies can change without redeploy. The PDP evaluates
    token claims + patient relationship + consent state + action → allow/deny. The PEP (gateway/filter) calls it
    on every request. Auditors love it — policy is reviewable, versioned, and testable.

---

## 11. Mental Cheat-Sheet

> **Three layers, all mandatory:**
>
> 1. **AuthN** — OIDC/OAuth2 with MFA, SMART on FHIR for interop.
> 2. **AuthZ** — RBAC + ABAC + relationship + consent, evaluated per request by a PDP.
> 3. **Audit** — append-only, hash-chained, tamper-evident log on every access.

```
   THE FOUR QUESTIONS A HEALTHCARE AUTH SYSTEM MUST ANSWER PER REQUEST

   1. WHO          is the user? (authenticated identity, role, org, tenant)
   2. WHAT         are they trying to do? (action + purpose_of_use)
   3. WHOSE        data is it? (patient + resource)
   4. DO WE HAVE CONSENT?    (consent registry, current version)

   ───────────────────────────────────────────────────────
   If any answer is missing or "no" → DENY + AUDIT.
   If all yes → ALLOW + AUDIT.
   Break-glass → ALLOW + AUDIT + ALERT.
```

### Compliance check before shipping

- [ ] Unique user IDs, no shared accounts.
- [ ] MFA enforced for clinicians, step-up for high-risk.
- [ ] Automatic logoff / short token TTL.
- [ ] Encryption at rest (KMS) + in transit (TLS 1.3) + per-field for sensitive categories.
- [ ] Audit log: append-only, hash-chained, 6+ year retention.
- [ ] Consent capture per-purpose; revocation propagates to tokens in < 1 min.
- [ ] Minimum-necessary enforced by row + field security.
- [ ] Break-glass flow with reason capture + 24–72h review.
- [ ] BAAs in place with every vendor touching PHI.
- [ ] Data subject rights implemented (access, rectification, restricted erasure, portability).
- [ ] Risk assessment documented; policies current; workforce trained.
- [ ] Breach notification runbook that hits HIPAA 60d and GDPR 72h.

### Related Topics

- `Java/spring/security/` — Spring Security + OAuth2 implementation
- `System-Design/API-Design/rest-principles.md` — REST + FHIR
- `Frontend/Browser-Internals/web-storage-and-cookies.md` — token storage on the client
- `Databases/Transactions/transactions-and-isolation-levels.md` — consistency for consent writes
- `Engineering-Practices/Design-Patterns/` — for the policy/PDP/PEP architecture
