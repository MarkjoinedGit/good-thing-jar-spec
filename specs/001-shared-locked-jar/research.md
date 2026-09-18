# Phase 0 Research: Shared Locked Jar

## 1. Runtime baseline

**Decision**: Use Maven coordinates `com.goodthingjar:good-thing-jar-backend:0.0.1-SNAPSHOT`, the
`com.goodthingjar` base package, Java 21 LTS, Spring Boot 4.1.1, Spring Security and Spring Data JPA
from the Boot dependency management, PostgreSQL 18, and Maven Wrapper 3.9.16. Keep planning
artifacts in `C:\workspace\good-thing-jar\good-thing-jar-spec` and apply all application, build,
migration, and Compose changes in `C:\workspace\good-thing-jar\good-thing-jar-backend`.

**Rationale**: The initialized backend POM confirms the authoritative group ID, artifact ID,
`java.version` 21, Spring Boot 4.1.1 parent, and Maven 3.9.16 wrapper. Its generated launcher and
context test require normalization to the authoritative `com.goodthingjar` root before feature
packages are added. Spring Boot 4.1.1 supports Java 21, so the plan follows the existing build
baseline without requiring a Java runtime change. Using the Boot dependency platform prevents
incompatible hand-picked Spring versions. PostgreSQL is the requested durable consistency boundary
and supports the constraints and transactional behavior needed by pairing and jar creation. The two
sibling repositories are separate roots; the Maven project and source tree belong to the backend
repository.

**Alternatives considered**:

- Java 25 LTS: supported by Spring Boot 4.1.1, but it would change the Java 21 baseline already
  declared by the backend project.
- Spring Boot 3.x: stable but superseded for a new project; no compatibility constraint requires it.
- Multiple deployables: adds distributed consistency and operations work without a product need.

**Sources**:

- [Spring Boot system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [PostgreSQL unique indexes](https://www.postgresql.org/docs/18/indexes-unique.html)

## 2. Modular monolith boundaries

**Decision**: Build one deployable with four business modules:
`com.goodthingjar.identity`, `com.goodthingjar.pairing`, `com.goodthingjar.jar`, and
`com.goodthingjar.notification`, with shared technical policy in `com.goodthingjar.platform`. Use
package visibility and Spring Modulith verification to prevent cycles and access to module internals.

**Rationale**: These modules follow distinct ownership boundaries while retaining one transaction
manager and database. Public module APIs exchange IDs and immutable DTOs; JPA entities never cross a
module boundary. Spring Modulith verifies acyclic dependencies and access through API packages. The
modules share the authoritative application root and do not introduce a second backend project.

**Alternatives considered**:

- Traditional technical layers across the whole service: permits unrelated features to couple
  through repositories and entities.
- Separate microservices: makes pair acceptance and initial jar creation distributed without a
  scale or organizational reason.
- Package conventions without verification: simple initially, but boundary drift would be silent.

**Source**:

- [Spring Modulith module verification](https://docs.spring.io/spring-modulith/reference/verification.html)

## 3. Lock status and trusted time

**Decision**: Persist only `effectiveUnlockAt` as an absolute instant plus the jar's IANA time-zone
snapshot. Derive LOCKED as `clock.instant() < effectiveUnlockAt` and UNLOCKED otherwise. Inject
`java.time.Clock` into application services and compute the default instant as local January 1
00:00:00 of the next calendar year converted through the jar's fixed zone.

**Rationale**: This exactly matches the specification, removes scheduler lag and split-brain state,
and makes boundary behavior deterministic in tests. Store instants in UTC-capable PostgreSQL
timestamps and retain the zone ID for display and future default calculations.

**Alternatives considered**:

- Persisting LOCKED/UNLOCKED and scheduling a transition: can become stale and creates a second
  source of truth.
- Trusting client time: allows early access by manipulating a request.
- Storing only a local date-time: produces ambiguous instants around offsets and daylight saving.

## 4. Transaction and concurrency policy

**Decision**: Put transactions on application use cases. Lock invitation/account rows in stable ID
order when accepting an invitation; lock the pair row when creating the next jar; lock the proposal
and jar when resolving a date change. Use database uniqueness constraints as final invariant guards.
Entry writes take a shared pessimistic lock on the jar, compare one injected-clock instant, and then
insert, while proposal approval takes an exclusive lock on that jar. Invitation creation first marks
any matching invitation whose seven-day deadline has passed as `EXPIRED`, then inserts under a
partial unique index on `(inviter_account_id, target_email_normalized)` for the stable active statuses
`PENDING_DELIVERY`, `PENDING`, and `DELIVERY_FAILED`. Invitation acceptance and cancellation lock the
same invitation row; conditional delivery-result updates cannot change `CANCELLED` or `EXPIRED` back
to an active status.

**Rationale**: The application checks produce clear domain errors, while database constraints make
concurrent duplicates impossible. Stable lock order limits deadlocks. Shared locks allow concurrent
writes but serialize them against changes to the effective unlock instant. Spring Data JPA supports
explicit lock metadata, and outer service transactions define the real unit of work.

**Alternatives considered**:

- Application checks without database constraints: race-prone.
- Global serializable isolation: broad contention and retry complexity for unrelated requests.
- A current-time partial index for locked jars: PostgreSQL index predicates cannot depend on
  changing current time; a stable `current` lifecycle marker plus a locked pair row is used for
  next-jar creation instead. The marker selects the active writing-cycle record but never determines
  LOCKED/UNLOCKED behavior.
- A current-time partial index for active invitations: the predicate cannot depend on `expires_at`.
  Stable active statuses form the predicate, and the creation transaction expires stale matches
  before insertion; the unique index remains the final guard against concurrent duplicate creation.
- Optimistic retries alone: viable, but row locks make these low-frequency transitions clearer.

**Sources**:

- [Spring Data JPA locking](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [Spring Data JPA transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)
- [PostgreSQL partial indexes](https://www.postgresql.org/docs/18/indexes-partial.html)

## 5. Authentication and logout

**Decision**: Use email/password authentication with adaptive password hashing and random opaque
bearer access/refresh tokens. Store only token hashes in PostgreSQL. Authenticate each request through
Spring Security against an active, unexpired server-side session. Rotate refresh tokens and revoke the
whole session on logout or detected reuse.

**Rationale**: Server-side sessions satisfy immediate logout invalidation for both web and mobile
REST clients. Opaque tokens avoid placing personal or authorization data in client-readable claims.
Short access-token lifetime and refresh rotation limit exposure while preserving usability.

**Alternatives considered**:

- Self-contained JWT access tokens without a revocation check: an ended session remains usable until
  expiry, contradicting the sign-out requirement.
- HTTP Basic on every request: repeatedly exposes a long-lived credential and provides no session
  revocation semantics.
- Browser cookies only: secure for web, but less uniform for first-party mobile REST clients.

**Source**:

- [Spring Security authentication architecture](https://docs.spring.io/spring-security/reference/servlet/authentication/)

## 6. Authorization and existence privacy

**Decision**: Enforce membership in application services and use requester-scoped repository queries.
For protected IDs, normalize nonexistent and unauthorized results to the same external problem type,
status, fields, and message. Check membership before checking lock state or returning metadata.

**Rationale**: Controllers alone are an insufficient boundary because use cases can be called from
tests and future adapters. Scoped lookup and normalized errors prevent direct-ID enumeration. An
authorized partner may receive a distinct `JAR_LOCKED` error because membership is already proven.

**Alternatives considered**:

- Returning different forbidden and not-found responses: leaks resource existence.
- Filtering only in controllers: duplicates policy and risks bypass through non-HTTP callers.
- Hiding lock status from members: unnecessary and conflicts with jar settings visibility.

## 7. Entry write and read contracts

**Decision**: Return `204 No Content` after accepting an entry, with no entry ID, `Location`, text,
count, or timestamps. Expose entry reads only as an authorized, cursor-paginated jar collection after
unlock. Order by `(createdAt, id)` and encode both fields in an opaque cursor.

**Rationale**: The write response confirms success without creating a read surface. Stable keyset
pagination supports jars with 10,000 or more entries and prevents gaps or duplicates across pages.
The tie-breaker handles equal timestamps.

**Alternatives considered**:

- Returning the created entry DTO: violates the pre-unlock read rule.
- Offset pagination: increasingly expensive and less stable if data changes.
- A direct entry-detail endpoint: unnecessary for the specified retrieval flow and expands the
  existence-disclosure surface.

## 8. Persistence mapping and migrations

**Decision**: Keep JPA entities internal, use scalar UUID foreign-key values across modules, default
all to-one mappings to lazy loading, avoid to-many entity collections for entries/invitations, and
use explicit projections or fetch joins per query. Create every table, constraint, and index through
ordered Flyway SQL migrations; disable automatic schema creation outside tests.

**Rationale**: Explicit queries make access cost visible and prevent accidental persistence-context
behavior and N+1 access. Flyway supplies an auditable schema history and runs PostgreSQL-compatible
migrations transactionally when possible.

**Alternatives considered**:

- Exposing JPA entities through controllers: couples the contract to persistence and leaks fields.
- Bidirectional mappings across modules: creates cycles, large graphs, and implicit queries.
- ORM-generated production schema: lacks controlled, reviewable migration history.

**Source**:

- [Flyway migration transaction handling](https://documentation.red-gate.com/fd/migration-transaction-handling-273973399.html)

## 9. Email delivery and invitation lifecycle

**Decision**: Create an invitation with status `PENDING_DELIVERY` and its sanitized delivery message
in a PostgreSQL-backed outbox in one transaction. `POST /invitations` returns `202 Accepted` with
that status without waiting for SMTP. For an invitation that remains active and unexpired, the
notification module changes it to `PENDING` after successful delivery or `DELIVERY_FAILED` after a
failed delivery. The inviter can inspect the
outgoing invitation and request a retry, which returns the same invitation to `PENDING_DELIVERY` and
queues delivery without creating another invitation. Only `PENDING` may be accepted; pair and
initial-jar creation occur atomically during acceptance by the verified target account. Each worker
outcome updates the outbox record and invitation consistently in one database transaction.
After an invitation reaches `DELIVERY_FAILED`, the failed outbox attempt is not automatically
rescheduled; only the authenticated delivery-retry operation queues a new attempt for that invitation.
`expires_at` is fixed at exactly seven days after `created_at`; delivery and retry never change it.
Delivery workers skip expired or cancelled invitations. Result handling uses a conditional update
that applies only while the invitation remains in the expected active status, so an external handoff
that races after cancellation cannot reactivate it. Acceptance and cancellation lock the same row;
only one of the mutually exclusive terminal outcomes `ACCEPTED` or `CANCELLED` can commit.

The active-invitation partial unique index uses only stable statuses and covers inviter plus normalized
target email. Because PostgreSQL index predicates cannot depend on changing time, creation locks the
matching active row, records it as `EXPIRED` when its deadline has passed, and then attempts insertion.
The unique index converts concurrent duplicate creation into one success and one normalized conflict.
After any terminal status, a new invitation may be created if the inviter remains eligible.

Email verification also uses the outbox. Verification tokens expire 24 hours after issuance; the
verification record stores only a one-way token hash. The outbox stores the delivery secret only in
encrypted, access-controlled form and clears it after terminal processing. The encryption key comes
from the deployment environment or secret manager and is never stored with the outbox data or in the
repository. The mail-delivery path decrypts the secret only to compose the intended verification
message to the account owner; that message is
the only permitted output containing the verification secret. API responses, logs, metrics, events,
errors, and audit records never contain it. For an unverified account, resend locks the account,
supersedes every prior unsuperseded token, including expired tokens, and creates exactly one newest
token and outbox message. Every request
returns the same `202 Accepted` response for absent, unverified, and already verified email addresses.
A partial unique index on account for unsuperseded, unconsumed verification records is the final
concurrency guard. Resend never updates an invitation or its expiry.

**Rationale**: An outbox avoids losing email after a database commit and avoids holding a database
transaction open during network I/O. Delivery records contain tokens only in one-time protected form
and never contain jar entries. The specification's retryable failure is the observable
`DELIVERY_FAILED` status plus the retry operation, rather than a synchronous SMTP error from the
original request. The same accepted response shape is used regardless of whether the target email
already belongs to an account, preserving account-enumeration protection. Conditional state changes
allow cancellation to complete without holding a database lock across SMTP and ensure late delivery
results cannot revive terminal invitations.

**Alternatives considered**:

- Sending SMTP inside the invitation transaction: holds locks across network latency and cannot
  atomically coordinate an external mail server with PostgreSQL.
- Requiring a message broker: adds infrastructure that the initial workload does not require.
- Fire-and-forget email: cannot meet the retryable failure behavior.
- A partial unique index whose predicate checks `expires_at > now()`: PostgreSQL requires immutable
  index predicates, so expiry must be materialized into a stable terminal status before insertion.

## 10. Testing and observability

**Decision**: Use pure unit tests with a fixed/mutable `Clock` for time rules; PostgreSQL
Testcontainers for migrations, constraints, locks, and concurrency; MockMvc and Spring Security Test
for REST/auth behavior; Spring Modulith verification for module boundaries. Expose health, readiness,
request latency, failure counts, outbox backlog, and security-event counts through Actuator/Micrometer.

**Rationale**: H2 or mocked repositories cannot prove PostgreSQL concurrency and index behavior.
Behavior-focused tests directly cover the highest-risk requirements. Operational signals diagnose
failures without logging entry content, passwords, tokens, invitation secrets, or raw email addresses.

**Alternatives considered**:

- Unit tests only: cannot validate migrations, queries, or concurrent transactions.
- Full-context tests for every rule: slower and less precise than a balanced test pyramid.
- Request/response body logging: creates unacceptable sensitive-data exposure.

**Source**:

- [Spring Boot service connections and Testcontainers](https://docs.spring.io/spring-boot/reference/features/dev-services.html)

## 11. External API description

**Decision**: Define the REST contract in OpenAPI 3.1.2 and treat it as a reviewed design artifact.
Use versioned `/api/v1` paths, JSON request/response DTOs, RFC 9457-style problem details, ISO-8601
instants, and IANA time-zone IDs.

**Rationale**: OpenAPI is language-neutral and allows client and contract tests to reason about the
service without seeing implementation code. Versioned paths preserve room for compatible evolution.

**Alternatives considered**:

- Code-only controller documentation: cannot be reviewed before implementation.
- GraphQL: adds schema/runtime complexity without a query-shape need.
- OpenAPI 3.2: newer but has less mature tooling; 3.1.2 meets the contract needs.

**Source**:

- [OpenAPI Specification 3.1.2](https://spec.openapis.org/oas/v3.1.2.html)

## 12. Shared abuse throttling

**Decision**: Implement configurable fixed-window throttles in PostgreSQL for authentication attempts,
email-verification resend, invitation creation and delivery retry, and entry writes. Store one row per
operation, keyed pseudonymous scope hash, and window start; update counters atomically and calculate the
`Retry-After` delay from the trusted clock and window end. Each operation has separate capacity and
window settings. Evaluate throttling before protected-resource lookup where needed, and use the same
429 problem shape and retry header regardless of account, email, invitation, membership, or resource
existence. Expired buckets are deleted as maintenance work.

Unauthenticated authentication and resend scopes combine a coarse client-source bucket with a keyed
normalized-email value that can be computed whether or not the account exists. Authenticated
invitation and entry operations use the authenticated account scope; session refresh uses keyed
presented-token material plus the client-source bucket. Invitation retry never requires resource
lookup to decide its throttle response. Raw scope inputs are not stored or logged.

**Rationale**: PostgreSQL is already shared by every horizontally repeatable application instance, so
database-backed counters enforce one limit across instances without adding Redis, a broker, gateway,
or process-local state. Fixed windows are simple to configure and test. They limit request bursts but
do not count accepted entries by day, impose a daily quota, or permanently reduce entry capacity. A
throttled entry attempt is not stored and can be retried after the returned delay; the 100-entry
acceptance test can stay within the configured rate or honor each retry delay.

**Alternatives considered**:

- In-memory counters: application instances would enforce different limits and restarts would reset
  them unpredictably.
- Redis, a message broker, or an API gateway: adds a deployment dependency solely for this feature.
- A daily entry counter: contradicts unlimited daily writing and the acceptance criterion.
