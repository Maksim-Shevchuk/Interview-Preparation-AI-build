# Interview Preparation

Personal knowledge base for technical interviews.

**Profile:** Java Full-Stack Developer (React + Spring Framework).
**Goal:** Cover Computer Science fundamentals, system design, language/framework specifics, and behavioral topics through structured Markdown notes.

---

## Architecture

The repository is organized into 8 top-level sections. Each section groups related topics; each topic lives in its own folder or `.md` file.

Legend: `✅` = note written · unmarked = planned placeholder.

```
Interview-Preparation/
├── README.md                            ← this file (architecture index)
│
├── Meta/                                ← planning + soft skills
│   ├── learning-path.md
│   ├── weekly-plan.md
│   ├── progress-checklist.md
│   ├── STAR-method.md
│   ├── leadership-principles.md
│   ├── common-questions.md
│   ├── salary-negotiation.md
│   └── company-specific/
│
├── CS-Fundamentals/                     ← core Computer Science theory
│   ├── Data-Structures/
│   ├── Algorithms/
│   ├── Complexity-Analysis/
│   ├── Mathematics-for-CS/
│   ├── Operating-Systems/
│   ├── Computer-Networks/
│   └── Distributed-Systems/
│
├── System-Design/                       ← interview-focused system design
│   ├── High-Level-Design/
│   │   ├── scaling-patterns.md
│   │   ├── caching-strategies.md
│   │   ├── load-balancing.md
│   │   ├── database-sharding.md
│   │   ├── message-queues.md
│   │   └── case-studies/                ← Twitter, Uber, Netflix, WhatsApp
│   ├── Low-Level-Design/
│   │   ├── oop-design-principles.md
│   │   ├── design-patterns/
│   │   └── problems/                    ← Parking Lot, LRU Cache, URL Shortener
│   └── API-Design/
│       ├── rest-principles.md
│       ├── graphql.md
│       └── api-versioning.md
│
├── Java/                                ← Java language + Spring + Java-specific backend
│   ├── core/                            ← OOP, Collections, Generics, Exceptions
│   │   └── collections-hashmap-internals.md  ✅
│   ├── modern-java/                     ← Streams, Lambdas, Records, Sealed, Virtual Threads
│   │   └── stream-api.md                ✅
│   ├── jvm-internals/                   ← Memory model, GC, JIT, ClassLoader
│   │   └── memory-model-and-gc.md       ✅
│   ├── concurrency/                     ← Threads, Executors, Locks, CompletableFuture
│   │   ├── synchronized-and-volatile.md ✅
│   │   └── executors-and-thread-pools.md ✅
│   ├── spring/
│   │   ├── core/                        ← IoC, DI, AOP, Bean lifecycle
│   │   ├── boot/                        ← Auto-config, Starters, Actuator
│   │   ├── data/                        ← JPA, Hibernate, Transactions
│   │   │   ├── transactional-annotation.md  ✅
│   │   │   └── n-plus-one-problem.md        ✅
│   │   ├── security/                    ← Spring Security + AuthN/AuthZ (OAuth2, OIDC, JWT)
│   │   └── cloud/                       ← Microservices patterns
│   ├── rest-api/                        ← Spring MVC, WebFlux
│   ├── messaging/                       ← Spring Kafka, Spring AMQP
│   ├── testing/                         ← JUnit 5, Mockito, Testcontainers
│   ├── build-tools/                     ← Maven, Gradle
│   └── interview-questions.md
│
├── Frontend/                            ← JS/TS, React, and the web platform
│   ├── JavaScript/                      ← ES6+, event-loop, closures, prototypes
│   ├── TypeScript/                      ← type-system, generics, advanced-types
│   ├── React/
│   │   ├── fundamentals/
│   │   ├── hooks/
│   │   ├── state-management/
│   │   ├── performance/
│   │   ├── patterns/
│   │   ├── testing/
│   │   └── ssr-and-rsc/
│   ├── HTML-CSS/
│   ├── Browser-Internals/               ← + web-security.md (OWASP, XSS, CSRF, CORS, CSP)
│   ├── Web-Performance/
│   ├── Accessibility/
│   └── Build-Tools/                     ← Vite, Webpack, esbuild
│
├── Databases/                           ← cross-cutting database knowledge
│   ├── SQL/                             ← queries, joins, window functions
│   ├── NoSQL/                           ← MongoDB, Cassandra, DynamoDB
│   ├── Redis/                           ← caching patterns
│   ├── Transactions/                    ← ACID, isolation levels
│   ├── Indexing/
│   ├── Replication-and-Sharding/
│   └── CAP-Theorem/
│
├── DevOps-Cloud/                        ← infrastructure and operations
│   ├── Docker/
│   ├── Kubernetes/
│   ├── CI-CD/                           ← GitHub Actions, Jenkins
│   ├── AWS/                             ← EC2, S3, RDS, Lambda
│   ├── Linux/
│   ├── Monitoring/                      ← Prometheus, Grafana, ELK
│   └── Infrastructure-as-Code/          ← Terraform, Ansible
│
└── Engineering-Practices/               ← software engineering methodology
    ├── OOP/
    ├── SOLID/
    ├── Design-Patterns/                 ← GoF + architectural
    ├── Clean-Code/
    ├── Refactoring/
    ├── Testing-Strategies/              ← Unit, Integration, E2E, TDD
    ├── Git/
    └── Agile-Methodologies/
```

---

## Section Guide

### Meta
Planning documents and soft-skill preparation. Learning paths, weekly plans, progress checklists. Behavioral interview content: STAR method, leadership principles, common questions, salary negotiation, company-specific notes.

### CS-Fundamentals
Core Computer Science theory — the academic foundation. Data structures, algorithms, complexity analysis, math for CS, operating systems, computer networks, and distributed systems theory (CAP, consensus, consistency models).

### System-Design
Interview-focused system design content. **High-Level Design:** scaling patterns, caching, load balancing, sharding, message queues, real-world case studies. **Low-Level Design:** OOP principles, design patterns, classic problems (Parking Lot, LRU Cache, URL Shortener). **API Design:** REST, GraphQL, versioning.

### Java
Everything Java-related, including the Spring ecosystem and Java-specific backend topics. Language core, modern features, JVM internals, concurrency, Spring (Core, Boot, Data, Security, Cloud), REST API, messaging, testing, build tools. Spring Security also hosts general AuthN/AuthZ concepts (OAuth2, OIDC, JWT) used in this stack.

**Currently covered (Tier 1+2 for Middle Full-Stack):**
- `core/collections-hashmap-internals.md` — HashMap structure, put/get, resize, treeification, ConcurrentHashMap
- `modern-java/stream-api.md` — pipeline, intermediate/terminal ops, collectors, parallel streams, pitfalls
- `jvm-internals/memory-model-and-gc.md` — memory areas, heap generations, GC algorithms, G1/ZGC/Shenandoah
- `concurrency/synchronized-and-volatile.md` — JMM, happens-before, monitors, volatile, double-checked locking
- `concurrency/executors-and-thread-pools.md` — ExecutorService, ThreadPoolExecutor, sizing, ForkJoinPool, CompletableFuture
- `spring/data/transactional-annotation.md` — propagation, isolation, rollback rules, self-invocation, reactive
- `spring/data/n-plus-one-problem.md` — OSIV, JOIN FETCH, @EntityGraph, @BatchSize, DTO projections

### Frontend
JavaScript, TypeScript, React, and the broader web platform. Browser internals also covers web security topics (OWASP Top 10, XSS, CSRF, CORS, CSP). Build tools covers Vite, Webpack, esbuild.

### Databases
Cross-cutting database knowledge. SQL and NoSQL engines, Redis for caching, transactions, indexing, replication, sharding, CAP theorem. ORM-specific notes (JPA/Hibernate) live in `Java/spring/data/`.

### DevOps-Cloud
Infrastructure and operations. Containers, orchestration, CI/CD, AWS, Linux, monitoring, infrastructure as code.

### Engineering-Practices
Software engineering methodology. OOP, SOLID, design patterns, clean code, refactoring, testing strategies, Git, agile.

---

## Topic Boundaries

Some topics span multiple sections. To avoid duplication, the following table defines which section "owns" each topic:

| Topic | Theory / Concepts | Applied / Framework-specific |
|-------|-------------------|------------------------------|
| Caching | `CS-Fundamentals/Distributed-Systems/` | `System-Design/High-Level-Design/caching-strategies.md`, `Databases/Redis/` |
| Concurrency | `CS-Fundamentals/Operating-Systems/` | `Java/concurrency/` |
| Transactions (ACID) | `Databases/Transactions/` | `Java/spring/data/` (JPA/Hibernate) |
| Design Patterns | `Engineering-Practices/Design-Patterns/` | `System-Design/Low-Level-Design/design-patterns/` |
| OOP | `Engineering-Practices/OOP/` | `System-Design/Low-Level-Design/oop-design-principles.md` |
| Distributed systems | `CS-Fundamentals/Distributed-Systems/` | `System-Design/High-Level-Design/` |
| HTTP / TLS / Web protocols | `CS-Fundamentals/Computer-Networks/` | `Frontend/Browser-Internals/` |
| AuthN / AuthZ | `Java/spring/security/` (concepts) | `Java/spring/security/` (Spring), `System-Design/HLD/` (at scale) |
| Web security (XSS, CSRF, CORS, CSP) | — | `Frontend/Browser-Internals/web-security.md` |
| Event loop | `Frontend/JavaScript/` (language level) | `Frontend/Browser-Internals/` (rendering pipeline) |

When a note is added, link to related notes in a `## Related` section rather than duplicating content.

---

## Conventions

- **Folder names:** `PascalCase` for multi-word folders (e.g., `Data-Structures`, `System-Design`). Single-word folders lowercase (e.g., `Java`, `Databases`).
- **File names:** `kebab-case.md` (e.g., `garbage-collection.md`, `big-o-notation.md`).
- **One topic per file:** Each `.md` file covers a single concept. Split large topics into multiple files.
- **Note structure** (recommended template):
  1. Brief theory
  2. Key concepts / definitions
  3. Code examples
  4. Common interview questions
  5. `## Related` — links to overlapping topics
  6. `## Resources` — books, articles, videos
- **Empty folder tracking:** Each empty folder has a `.gitkeep` file so git tracks the structure. Delete the `.gitkeep` when adding the first real note.
- **Language:** All note content is written in English.

---

## Progress Tracking

- [ ] Meta
- [ ] CS-Fundamentals
- [ ] System-Design
- [~] Java — **7 notes written** (Tier 1+2 covered; see Section Guide above)
- [ ] Frontend
- [ ] Databases
- [ ] DevOps-Cloud
- [ ] Engineering-Practices

Legend: `[ ]` not started · `[~]` in progress · `[x]` interview-ready.

### Java — Next Priorities (Tier 2)

- `Java/spring/core/bean-lifecycle.md` — IoC, DI, Bean lifecycle, scopes, AOP
- `Java/rest-api/error-handling.md` — `@ControllerAdvice`, RFC 7807, validation, idempotency
- `Java/spring/boot/autoconfiguration.md` — auto-config, starters, actuator
- `Java/testing/junit-and-mockito.md` — JUnit 5, Mockito, Testcontainers
- `Java/core/equals-and-hashcode.md` — contract deep dive

---

## Future Expansion

- **Kotlin/** — to be added as a peer of `Java/` when needed. Planned structure: `core/`, `coroutines/`, `spring/` (Kotlin-first Spring), `interop-with-java.md`, and optionally `android/` or KMP-related folders.
- Additional languages (Go, Python) can be added as peer top-level folders following the `Java/` pattern.
- New topics within existing sections can be added freely; just follow the conventions above.
