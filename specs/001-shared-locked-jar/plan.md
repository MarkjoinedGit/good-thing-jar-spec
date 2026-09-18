# Implementation Plan: Shared Locked Jar

**Specification Branch**: `001-shared-locked-jar` | **Date**: 2026-09-21 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from
`C:\workspace\good-thing-jar\good-thing-jar-spec\specs\001-shared-locked-jar\spec.md`

## Summary

Build a production-ready REST backend that lets two verified accounts form one private pair, write
unlimited entries into one current jar, and read those entries only when trusted system time reaches
the jar's effective unlock instant. Implement this as a single Spring Boot deployable organized into
verified application modules for identity, pairing, jars, and notification delivery. PostgreSQL is
the consistency boundary; application services own transactions, Flyway owns schema evolution, and
database constraints plus explicit locking protect pairing, proposal, and next-jar races.
Invitation creation atomically stores a `PENDING_DELIVERY` invitation and PostgreSQL outbox message;
eligible delivery later changes it to `PENDING` or `DELIVERY_FAILED`, and only an unexpired `PENDING`
invitation can be accepted.
Stable-status uniqueness plus row locking prevents duplicate active invitations, while terminal
`ACCEPTED`, `CANCELLED`, `EXPIRED`, and `INVALIDATED` states cannot return to an active state.
Email-verification replacement uses the same outbox, supersedes older tokens, and leaves invitation
expiry unchanged. PostgreSQL-backed, operation-specific short-window throttle buckets provide one
shared abuse-control boundary across horizontally repeatable application instances.

## Technical Context

**Language/Version**: Java 21 LTS
**Primary Dependencies**: Spring Boot 4.1.1, Spring Web MVC, Spring Security 7.1.x, Spring Data JPA 4.1.x, Spring Modulith 2.1.1, Bean Validation, Flyway, PostgreSQL JDBC, Spring Boot Actuator
**Build Tool**: Maven Wrapper 3.9.16
**Maven Coordinates**: `com.goodthingjar:good-thing-jar-backend:0.0.1-SNAPSHOT`
**Base Package**: `com.goodthingjar`
**Java Source Root**: `src/main/java/com/goodthingjar/`
**Test Source Root**: `src/test/java/com/goodthingjar/`
**Specification Root**: `C:\workspace\good-thing-jar\good-thing-jar-spec`
**Implementation Root**: `C:\workspace\good-thing-jar\good-thing-jar-backend`
**Storage**: PostgreSQL 18 with Flyway versioned SQL migrations and platform-managed encryption at rest
**Testing**: JUnit Jupiter through Spring Boot Test, Spring Security Test, MockMvc, Spring Modulith
module verification, Testcontainers PostgreSQL, and deterministic `Clock` tests
**Target Platform**: Containerized Linux service with an external PostgreSQL database and SMTP
email delivery; horizontally repeatable application instances with no local session state
**Project Type**: Single deployable Spring Boot REST backend using a modular monolith architecture
**Performance Goals**: In the documented reference workload of 100 active pairs, p95 write
acknowledgement under 2 seconds and p95 time to first post-unlock entry page under 3 seconds
**Constraints**: Strict pre-unlock content and metadata denial; immediate logout revocation; no
persisted lock-state source of truth; exactly two accounts per pair; at most one current locked jar;
asynchronous outbox-backed invitation and verification delivery; acceptance only after confirmed
invitation delivery; required configurable throttles with retry delays and no daily entry quota; no
entry content in logs, events, metrics, token claims, or email; verification secrets appear only in
the intended account-owner verification message, never in API responses or diagnostic surfaces, and
exist in the outbox only as temporary encrypted, access-controlled delivery data
**Scale/Scope**: Initial release supports at least 10,000 entries per jar, 100 concurrently active
pairs in the acceptance workload, cursor-paginated reads, and one application/database deployment
unit. The 100-same-day-entry criterion runs within configured short-window rates or retries after
the returned delay; it does not require 100 simultaneous first attempts to succeed.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Pre-design evaluation

- **Correctness and compatibility — PASS**: The plan implements the finalized behavior without
  adding entry edits, deletion, pair dissolution, partner replacement, reminders, or an admin UI.
  Trusted time, pair membership, mutual approval, and one-current-jar invariants remain explicit.
- **Architecture and API — PASS**: Controllers expose request/response DTOs only. Application
  services own use cases and authorization decisions; persistence adapters and JPA entities remain
  module-internal. A centralized problem-details handler normalizes validation and protected-resource
  failures.
- **Persistence and transactions — PASS**: Pair acceptance, proposal resolution, and next-jar
  creation have explicit transactional boundaries. Invitation creation, cancellation, acceptance,
  expiry, delivery results, and retries use stable-state uniqueness and row locking. Database
  uniqueness constraints are final guards; deliberate row locking resolves races. Associations
  default to lazy access and paged queries avoid unbounded collections and N+1 reads. All schema
  changes use Flyway.
- **Security and observability — PASS**: Spring Security authenticates opaque bearer sessions whose
  secrets are stored only as hashes and can be revoked immediately. Authorization is checked at the
  service boundary. Shared PostgreSQL throttles cover authentication, verification resend,
  invitation creation/retry, and entry writes without existence disclosure or daily quotas. Logs,
  metrics, events, and errors exclude note text, secrets, raw tokens, and unnecessary personal data.
  Security denials and privileged operational access are auditable.
- **Testing and completion — PASS**: Unit tests cover time and domain rules; PostgreSQL integration
  tests cover constraints, migrations, locks, pagination, invitation transition races, verification
  replacement, and shared throttling; API tests cover contracts, retry delays, authorization, error
  equivalence, and absence of pre-unlock metadata. Relevant tests and module verification must pass.
- **Maintainability — PASS**: One Maven project in the backend repository and one deployable keep
  operations simple. Four cohesive application modules communicate through public module APIs and
  events. Spring Modulith
  is justified because it automatically verifies acyclic module boundaries in the modular monolith.

### Post-design re-evaluation

- **Correctness and compatibility — PASS**: The data model derives lock behavior solely from
  `effectiveUnlockAt` and an injected trusted `Clock`; the jar's `current` lifecycle marker is not
  lock state, and the API contract exposes no early-read path.
- **Architecture and API — PASS**: The OpenAPI contract uses dedicated schemas and the source tree
  keeps each module's `api`, `application`, `domain`, and `persistence` concerns distinct.
- **Persistence and transactions — PASS**: The design documents database constraints, lock order,
  stable invitation uniqueness, terminal-state guards, fetch plans, pagination, and migrations. No
  rule relies on an open persistence context.
- **Security and observability — PASS**: Protected-resource responses are normalized, sessions are
  revocable, and sensitive fields are excluded from telemetry and outbound events.
- **Testing and completion — PASS**: The quickstart defines automated verification using the real
  PostgreSQL engine; the research and data model identify the critical race and boundary tests.
- **Maintainability — PASS**: Dependencies follow `jar → pairing → identity`; notifications consume
  events without exposing module internals. No constitution exception is required.

## Project Structure

### Documentation (specification repository)

**Root**: `C:\workspace\good-thing-jar\good-thing-jar-spec`

```text
specs/001-shared-locked-jar/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── openapi.yaml
└── tasks.md                 # Created later by /speckit.tasks
```

### Source Code (implementation repository)

**Root**: `C:\workspace\good-thing-jar\good-thing-jar-backend`

```text
pom.xml
.mvn/
mvnw
mvnw.cmd
compose.yaml                                      # planned in this repository; not yet present
src/
├── main/
│   ├── java/com/goodthingjar/
│   │   ├── GoodThingJarBackendApplication.java  # authoritative application root
│   │   ├── identity/
│   │   │   ├── api/
│   │   │   ├── application/
│   │   │   ├── domain/
│   │   │   └── persistence/
│   │   ├── pairing/
│   │   │   ├── api/
│   │   │   ├── application/
│   │   │   ├── domain/
│   │   │   └── persistence/
│   │   ├── jar/
│   │   │   ├── api/
│   │   │   ├── application/
│   │   │   ├── domain/
│   │   │   └── persistence/
│   │   ├── notification/
│   │   │   ├── application/
│   │   │   └── infrastructure/
│   │   └── platform/
│   │       ├── error/
│   │       ├── security/
│   │       ├── throttle/
│   │       ├── time/
│   │       └── observability/
│   └── resources/
│       ├── application.properties               # existing configuration file
│       └── db/migration/
└── test/
    ├── java/com/goodthingjar/
    │   ├── GoodThingJarBackendApplicationTests.java
    │   ├── architecture/
    │   ├── identity/
    │   ├── pairing/
    │   ├── jar/
    │   ├── contract/
    │   └── integration/
    └── resources/
```

**Structure Decision**: Extend the existing Maven application in the sibling backend repository.
Use the authoritative `com.goodthingjar` base package for `GoodThingJarBackendApplication` and the
`com.goodthingjar.identity`, `com.goodthingjar.pairing`, `com.goodthingjar.jar`,
`com.goodthingjar.notification`, and `com.goodthingjar.platform` packages. Normalize the generated
launcher and context test to this root when implementation begins, before adding modules. Each
top-level business package is a Spring Modulith module. Its `api` package is the only public surface;
`application`, `domain`, and `persistence` packages are internal. Cross-module persistence
relationships use scalar identifiers rather than JPA entity references. The dependency direction is
`jar → pairing → identity`; pairing publishes pair-created events that the jar module handles in the
same transaction, and notification delivery consumes sanitized events. `platform` contains only
narrow technical policies shared by all modules and no business rules. Build, run, test, migration,
Compose, and source-generation commands execute from the implementation root; documentation edits
remain in the specification root.

## Complexity Tracking

No constitution violations require justification.
