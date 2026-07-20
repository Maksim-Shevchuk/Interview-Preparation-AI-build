# Interview Preparation

Personal knowledge base for technical interviews.

**Profile:** Java Full-Stack Developer (React + Spring Framework).
**Goal:** Cover Computer Science fundamentals, system design, language/framework specifics, and behavioral topics through structured Markdown notes.

Legend: ✅ note written · 📋 planned

---

## Table of Contents

### Meta 📋
Planning documents and soft-skill preparation.

- 📋 [Learning Path](./Meta/learning-path.md)
- 📋 [Weekly Plan](./Meta/weekly-plan.md)
- 📋 [Progress Checklist](./Meta/progress-checklist.md)
- 📋 [STAR Method](./Meta/STAR-method.md)
- 📋 [Leadership Principles](./Meta/leadership-principles.md)
- 📋 [Common Questions](./Meta/common-questions.md)
- 📋 [Salary Negotiation](./Meta/salary-negotiation.md)
- 📋 [Company-Specific](./Meta/company-specific/)

### CS-Fundamentals 📋
Core Computer Science theory — the academic foundation.

- 📋 [Data Structures](./CS-Fundamentals/Data-Structures/)
- 📋 [Algorithms](./CS-Fundamentals/Algorithms/)
- 📋 [Complexity Analysis](./CS-Fundamentals/Complexity-Analysis/)
- 📋 [Mathematics for CS](./CS-Fundamentals/Mathematics-for-CS/)
- 📋 [Operating Systems](./CS-Fundamentals/Operating-Systems/)
- 📋 [Computer Networks](./CS-Fundamentals/Computer-Networks/)
- 📋 [Distributed Systems](./CS-Fundamentals/Distributed-Systems/)

### System-Design 📋
Interview-focused system design content.

- 📋 [High-Level Design](./System-Design/High-Level-Design/)
  - 📋 [Scaling Patterns](./System-Design/High-Level-Design/scaling-patterns.md)
  - 📋 [Caching Strategies](./System-Design/High-Level-Design/caching-strategies.md)
  - 📋 [Load Balancing](./System-Design/High-Level-Design/load-balancing.md)
  - 📋 [Database Sharding](./System-Design/High-Level-Design/database-sharding.md)
  - 📋 [Message Queues](./System-Design/High-Level-Design/message-queues.md)
  - 📋 [Case Studies](./System-Design/High-Level-Design/case-studies/) — Twitter, Uber, Netflix, WhatsApp
- 📋 [Low-Level Design](./System-Design/Low-Level-Design/)
  - 📋 [OOP Design Principles](./System-Design/Low-Level-Design/oop-design-principles.md)
  - 📋 [Design Patterns](./System-Design/Low-Level-Design/design-patterns/)
  - 📋 [Problems](./System-Design/Low-Level-Design/problems/) — Parking Lot, LRU Cache, URL Shortener
- 📋 [API Design](./System-Design/API-Design/)
  - ✅ [REST Principles](./System-Design/API-Design/rest-principles.md)
  - 📋 [GraphQL](./System-Design/API-Design/graphql.md)
  - 📋 [API Versioning](./System-Design/API-Design/api-versioning.md)
- [Domain Design](./System-Design/Domain-Design/) — regulated & industry-specific
  - ✅ [Healthcare Auth — GDPR / HIPAA](./System-Design/Domain-Design/healthcare-auth-gdpr-hipaa.md)

### Java ✅ (in progress)
Java language + Spring ecosystem + Java-specific backend topics.

- [Core](./Java/core/) — OOP, Collections, Generics, Exceptions, Streams, Lambdas, Optional
  - ✅ [HashMap Internals](./Java/core/collections-hashmap-internals.md)
  - ✅ [Stream API](./Java/core/stream-api.md)
  - 📋 Equals & hashCode
  - 📋 Lambda Expressions
  - 📋 Optional
  - 📋 Records & Sealed Classes
- [Modern Java](./Java/modern-java/) — key changes per LTS version
  - ✅ [Java 8 LTS](./Java/modern-java/java-8-lts.md) — lambdas, Streams, Optional, java.time
  - ✅ [Java 11 LTS](./Java/modern-java/java-11-lts.md) — var, HTTP Client, String methods
  - ✅ [Java 17 LTS](./Java/modern-java/java-17-lts.md) — records, sealed, pattern matching, text blocks
  - ✅ [Java 21 LTS](./Java/modern-java/java-21-lts.md) — virtual threads, pattern matching for switch
  - ✅ [Java 25 LTS](./Java/modern-java/java-25-lts.md) — synchronized without pinning, scoped values, structured concurrency
- [JVM Internals](./Java/jvm-internals/) — Memory model, GC, JIT, ClassLoader
  - ✅ [Memory Model and GC](./Java/jvm-internals/memory-model-and-gc.md)
  - 📋 Class Loading
  - 📋 JIT Compilation
- [Concurrency](./Java/concurrency/) — Threads, Executors, Locks, CompletableFuture
  - ✅ [Thread Fundamentals](./Java/concurrency/thread-fundamentals.md)
  - ✅ [synchronized and volatile](./Java/concurrency/synchronized-and-volatile.md)
  - ✅ [The happens-before Relationship](./Java/concurrency/happens-before.md)
  - ✅ [Locks and Atomic](./Java/concurrency/locks-and-atomic.md)
  - ✅ [Concurrent Collections](./Java/concurrency/concurrent-collections.md)
  - ✅ [Synchronization Primitives](./Java/concurrency/synchronization-primitives.md)
  - ✅ [Executors and Thread Pools](./Java/concurrency/executors-and-thread-pools.md)
  - ✅ [CompletableFuture](./Java/concurrency/completable-future.md)
  - ✅ [Virtual Threads](./Java/concurrency/virtual-threads.md)
  - ✅ [Thread Tracing & Diagnostics](./Java/concurrency/thread-tracing-and-diagnostics.md)
- [Spring](./Java/spring/)
  - [Core](./Java/spring/core/) — IoC, DI, AOP, Bean lifecycle
    - ✅ [Spring Core](./Java/spring/core/spring-core.md) — IoC, DI, AOP, Bean lifecycle, scopes
  - [Boot](./Java/spring/boot/) — Auto-config, Starters, Actuator
    - ✅ [Spring Boot](./Java/spring/boot/spring-boot.md) — auto-config, starters, actuator
  - [Data](./Java/spring/data/) — JPA, Hibernate, Transactions
    - ✅ [@Transactional Deep Dive](./Java/spring/data/transactional-annotation.md)
    - ✅ [N+1 Problem](./Java/spring/data/n-plus-one-problem.md)
    - 📋 JPA & Hibernate Basics
    - 📋 Connection Pooling
  - 📋 [Security](./Java/spring/security/) — Spring Security + AuthN/AuthZ (OAuth2, OIDC, JWT)
  - [Cloud](./Java/spring/cloud/) — Microservices patterns
    - ✅ [Microservices Architecture](./Java/spring/cloud/microservices-architecture.md)
    - ✅ [Spring Cloud Ecosystem](./Java/spring/cloud/spring-cloud-ecosystem.md)
- [REST API](./Java/rest-api/) — Spring MVC, WebFlux
  - ✅ [Spring REST API](./Java/rest-api/spring-rest-api.md)
- [Messaging](./Java/messaging/) — Spring Kafka, Spring AMQP
  - ✅ [Apache Kafka](./Java/messaging/apache-kafka.md)
  - ✅ [RabbitMQ](./Java/messaging/rabbitmq.md)
- 📋 [Testing](./Java/testing/) — JUnit 5, Mockito, Testcontainers
- 📋 [Build Tools](./Java/build-tools/) — Maven, Gradle
- 📋 Interview Questions

### Frontend 📋
JavaScript, TypeScript, React, and the web platform.

- [JavaScript](./Frontend/JavaScript/) — ES6+, event-loop, closures, prototypes
  - ✅ [Types and Coercion](./Frontend/JavaScript/types-and-coercion.md)
  - ✅ [Closures and Scope](./Frontend/JavaScript/closures-and-scope.md)
  - ✅ [Prototypes and `this`](./Frontend/JavaScript/prototypes-and-this.md)
  - ✅ [Event Loop](./Frontend/JavaScript/event-loop.md)
  - ✅ [Promises and Async/Await](./Frontend/JavaScript/promises-and-async-await.md)
  - ✅ [ES6+ Features](./Frontend/JavaScript/es6-features.md)
  - ✅ [Property Descriptors](./Frontend/JavaScript/property-descriptors.md)
- [TypeScript](./Frontend/TypeScript/) — type-system, generics, advanced-types
  - ✅ [Type System Basics](./Frontend/TypeScript/type-system-basics.md)
  - ✅ [Interfaces vs Type Aliases](./Frontend/TypeScript/interfaces-vs-types.md)
  - ✅ [Generics](./Frontend/TypeScript/generics.md)
  - ✅ [Advanced Types](./Frontend/TypeScript/advanced-types.md)
  - ✅ [Type Narrowing](./Frontend/TypeScript/type-narrowing.md)
- [React](./Frontend/React/)
  - ✅ [Components, JSX, and Virtual DOM](./Frontend/React/fundamentals/components-jsx-and-virtual-dom.md)
  - ✅ [Synthetic Events — React vs Native](./Frontend/React/fundamentals/synthetic-events.md)
  - ✅ [Portals and Refs](./Frontend/React/fundamentals/portals-and-refs.md)
  - ✅ [Hooks in Depth](./Frontend/React/hooks/hooks-in-depth.md)
  - ✅ [State Management](./Frontend/React/state-management/state-management.md)
  - ✅ [Redux Toolkit — Deep Dive](./Frontend/React/state-management/redux-toolkit.md)
  - ✅ [Performance Optimization](./Frontend/React/performance/performance-optimization.md)
  - ✅ [Component Patterns](./Frontend/React/patterns/component-patterns.md)
  - ✅ [SSR, SSG, and RSC](./Frontend/React/ssr-and-rsc/ssr-ssg-and-rsc.md)
  - ✅ [React Testing](./Frontend/React/testing/react-testing.md)
- [HTML-CSS](./Frontend/HTML-CSS/)
  - ✅ [Semantic HTML](./Frontend/HTML-CSS/semantic-html.md)
  - ✅ [Box Model, Positioning, Selectors](./Frontend/HTML-CSS/box-model-positioning-selectors.md)
  - ✅ [Flexbox and Grid](./Frontend/HTML-CSS/flexbox-and-grid.md)
  - ✅ [Sass — Preprocessor & Code Generation](./Frontend/HTML-CSS/sass.md) — mixins, functions, @use modules, generated utilities
- [Browser Internals](./Frontend/Browser-Internals/) — + web-security (OWASP, XSS, CSRF, CORS, CSP)
  - ✅ [Web Storage and Cookies](./Frontend/Browser-Internals/web-storage-and-cookies.md)
  - ✅ [Web Workers and Service Workers](./Frontend/Browser-Internals/web-workers-and-service-workers.md)
- [Web Performance](./Frontend/Web-Performance/)
  - ✅ [Web Performance](./Frontend/Web-Performance/web-performance.md)
- [Accessibility](./Frontend/Accessibility/)
  - ✅ [Web Accessibility](./Frontend/Accessibility/web-accessibility.md)
- 📋 [Build Tools](./Frontend/Build-Tools/) — Vite, Webpack, esbuild
- [Testing](./Frontend/Testing/)
  - ✅ [Jest and React Testing Library](./Frontend/Testing/jest-and-react-testing-library.md)

### Databases 📋
Cross-cutting database knowledge.

- [SQL](./Databases/SQL/) — queries, joins, window functions
  - ✅ [SQL Fundamentals](./Databases/SQL/sql-fundamentals.md)
  - ✅ [EXPLAIN and Query Optimization](./Databases/SQL/explain-and-query-optimization.md)
- [NoSQL](./Databases/NoSQL/) — MongoDB, Cassandra, DynamoDB
  - ✅ [Performance Issues — Diagnosis and Resolution](./Databases/NoSQL/performance-issues.md)
- 📋 [Redis](./Databases/Redis/) — caching patterns
- [Transactions](./Databases/Transactions/) — ACID, isolation levels
  - ✅ [Transactions and Isolation Levels](./Databases/Transactions/transactions-and-isolation-levels.md)
- 📋 [Indexing](./Databases/Indexing/)
- 📋 [Replication and Sharding](./Databases/Replication-and-Sharding/)
- 📋 [CAP Theorem](./Databases/CAP-Theorem/)

### DevOps-Cloud 📋
Infrastructure and operations.

- [Docker](./DevOps-Cloud/Docker/)
  - ✅ [Docker Core Concepts](./DevOps-Cloud/Docker/docker-core-concepts.md)
- [Kubernetes](./DevOps-Cloud/Kubernetes/)
  - ✅ [Kubernetes Core Concepts](./DevOps-Cloud/Kubernetes/kubernetes-core-concepts.md)
- 📋 [CI/CD](./DevOps-Cloud/CI-CD/) — GitHub Actions, Jenkins
- 📋 [AWS](./DevOps-Cloud/AWS/) — EC2, S3, RDS, Lambda
- 📋 [Linux](./DevOps-Cloud/Linux/)
- 📋 [Monitoring](./DevOps-Cloud/Monitoring/) — Prometheus, Grafana, ELK
- 📋 [Infrastructure as Code](./DevOps-Cloud/Infrastructure-as-Code/) — Terraform, Ansible
- [SaaS](./DevOps-Cloud/SaaS/)
  - ✅ [SaaS — Software as a Service](./DevOps-Cloud/SaaS/saas.md) — multi-tenancy, billing, code generation

### Engineering-Practices 📋
Software engineering methodology.

- 📋 [OOP](./Engineering-Practices/OOP/)
- [SOLID](./Engineering-Practices/SOLID/)
  - ✅ [SOLID Principles](./Engineering-Practices/SOLID/solid-principles.md)
  - ✅ [GRASP Principles](./Engineering-Practices/SOLID/grasp-principles.md)
- 📋 [Design Patterns](./Engineering-Practices/Design-Patterns/) — GoF + architectural
- 📋 [Clean Code](./Engineering-Practices/Clean-Code/)
- 📋 [Refactoring](./Engineering-Practices/Refactoring/)
- 📋 [Testing Strategies](./Engineering-Practices/Testing-Strategies/) — Unit, Integration, E2E, TDD
- 📋 [Git](./Engineering-Practices/Git/)
- 📋 [Agile Methodologies](./Engineering-Practices/Agile-Methodologies/)

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

- 📋 Meta
- 📋 CS-Fundamentals
- 📋 System-Design
- ✅ Java — 15 notes written (Tier 1+2 covered; **Concurrency folder complete**, **Modern Java LTS versions complete**)
- 📋 Frontend
- 📋 Databases
- 📋 DevOps-Cloud
- 📋 Engineering-Practices

### Java — Next Priorities (Tier 2)

- 📋 `Java/spring/core/bean-lifecycle.md` — IoC, DI, Bean lifecycle, scopes, AOP
- 📋 `Java/rest-api/error-handling.md` — `@ControllerAdvice`, RFC 7807, validation, idempotency
- 📋 `Java/spring/boot/autoconfiguration.md` — auto-config, starters, actuator
- 📋 `Java/testing/junit-and-mockito.md` — JUnit 5, Mockito, Testcontainers
- 📋 `Java/core/equals-and-hashcode.md` — contract deep dive

---

## Future Expansion

- **Kotlin/** — to be added as a peer of `Java/` when needed. Planned structure: `core/`, `coroutines/`, `spring/` (Kotlin-first Spring), `interop-with-java.md`, and optionally `android/` or KMP-related folders.
- Additional languages (Go, Python) can be added as peer top-level folders following the `Java/` pattern.
- New topics within existing sections can be added freely; just follow the conventions above.
