# Phase 1 Quickstart: Shared Locked Jar

This guide describes the expected developer workflow after implementation. The planning artifacts
and Spring Boot application are in separate sibling repositories.

## Repository locations

- **Specification root**: `C:\workspace\good-thing-jar\good-thing-jar-spec`
- **Implementation root**: `C:\workspace\good-thing-jar\good-thing-jar-backend`
- **Maven coordinates**: `com.goodthingjar:good-thing-jar-backend:0.0.1-SNAPSHOT`
- **Base package**: `com.goodthingjar`
- **Java source root**: `src/main/java/com/goodthingjar/`
- **Test source root**: `src/test/java/com/goodthingjar/`

All build, run, test, migration, Docker Compose, and source-generation commands execute from the
implementation root. In PowerShell, establish that working directory first:

```powershell
$implementationRoot = 'C:\workspace\good-thing-jar\good-thing-jar-backend'
Set-Location -LiteralPath $implementationRoot
```

On a POSIX shell opened in `C:/workspace/good-thing-jar`:

```bash
cd good-thing-jar-backend
```

## Prerequisites

- JDK 21
- Docker with Compose support
- PowerShell 7 or a POSIX-compatible shell

No local PostgreSQL installation is required. Integration tests use Testcontainers, and the
implementation root includes `compose.yaml` for local PostgreSQL and Mailpit services.

## 1. Configure local secrets

Set development-only values through environment variables. Do not commit them.

```powershell
$env:SPRING_PROFILES_ACTIVE = 'local'
$env:GTJ_DB_URL = 'jdbc:postgresql://localhost:5432/good_thing_jar'
$env:GTJ_DB_USERNAME = 'good_thing_jar'
$env:GTJ_DB_PASSWORD = '<local-password>'
$env:GTJ_MAIL_HOST = 'localhost'
$env:GTJ_MAIL_PORT = '1025'
$env:GTJ_PUBLIC_BASE_URL = 'http://localhost:8080'
$env:GTJ_VERIFICATION_DELIVERY_KEY = '<local-development-encryption-key>'
$env:GTJ_THROTTLE_SCOPE_KEY = '<at-least-32-random-characters>'
```

Equivalent POSIX exports may be used. Production values must come from the deployment platform's
secret management rather than files in the repository.

## 2. Start local dependencies

```powershell
docker compose --file .\compose.yaml up -d postgres mailpit
docker compose --file .\compose.yaml ps
```

The Compose file should include PostgreSQL and a development SMTP inbox only. The application remains
a single process and deployable artifact.

## 3. Run the database migrations and application

```powershell
.\mvnw.cmd spring-boot:run
```

On POSIX systems:

```bash
./mvnw spring-boot:run
```

Startup must fail if Flyway cannot validate or migrate the schema. ORM schema creation must be
disabled. Verify health without exposing sensitive details:

```powershell
Invoke-RestMethod http://localhost:8080/api/v1/actuator/health
```

## 4. Run verification

Fast unit and architecture checks:

```powershell
.\mvnw.cmd test
```

Build the deployable artifact from the implementation repository:

```powershell
.\mvnw.cmd package
```

Full verification, including PostgreSQL integration, migration, REST contract, and concurrency tests:

```powershell
.\mvnw.cmd verify
```

The build must include these gates:

- Spring Modulith verification finds no cycles or access to module internals.
- Flyway migrates a clean PostgreSQL database and validates an upgraded database.
- Security tests prove revoked sessions fail immediately.
- Email-verification tests prove 24-hour expiry, generic resend responses, newest-token-only use under
  concurrent resend, reliable outbox delivery, unchanged invitation expiry, and absence of plaintext
  secrets from persistence, API responses, and diagnostic surfaces while allowing the intended
  account-owner verification message to contain the secret.
- PostgreSQL-backed throttle tests cover authentication, verification resend, invitation creation and
  delivery retry, and entry writes across multiple application instances. Every rejection includes
  `Retry-After` without existence disclosure or a daily entry quota.
- API tests prove unauthorized existing resources and nonexistent resources have the same external
  response.
- Fixed-clock tests deny reads and writes one instant before unlock and allow reads at the exact
  unlock instant without a scheduled transition.
- Concurrent invitation acceptance never places an account in two pairs.
- Invitation creation atomically records one `PENDING_DELIVERY` invitation and outbox message and
  returns without waiting for SMTP.
- Successful and failed delivery produce `PENDING` and `DELIVERY_FAILED`; retry reuses the same
  invitation and returns it to `PENDING_DELIVERY`.
- `PENDING_DELIVERY` and `DELIVERY_FAILED` invitations cannot create a pair or initial jar.
- Invitation tests prove exact seven-day expiry, sequential and concurrent duplicate rejection,
  cancellation from every active state, terminal-state creation eligibility, and cancellation races
  with acceptance, retry, and delivery results. No cancelled or expired invitation becomes active.
- Concurrent next-jar creation creates exactly one new jar.
- A locked entry submission response contains no ID, body, timestamp, author, count, or location.
- Query-count tests cover jar lists and entry pages to detect N+1 regressions.
- The 100-entry scenario submits within configured short-window rates or retries after `Retry-After`;
  every valid entry is eventually accepted without a daily quota.

## 5. Validate the API contract

The source contract remains in the specification repository at
[contracts/openapi.yaml](contracts/openapi.yaml). From the implementation root its sibling path is
`..\good-thing-jar-spec\specs\001-shared-locked-jar\contracts\openapi.yaml`. Backend build steps that
validate the contract or generate source must run through the backend Maven project, reference that
source path explicitly, and write generated output under the backend build tree.

Example flow after the application is running:

1. Register two accounts with `POST /api/v1/auth/registrations`.
2. Read verification emails in the local SMTP inbox and call
   `POST /api/v1/auth/email-verifications`.
3. Request a replacement with `POST /api/v1/auth/email-verification-resends`. Confirm absent,
   unverified, and already verified addresses receive the same `202 Accepted` response; for an
   unverified account, only the newest unexpired token succeeds and invitation expiry is unchanged.
4. Create sessions with `POST /api/v1/auth/sessions` and use each opaque access token as a bearer
   token.
5. Create an invitation with `POST /api/v1/invitations`, including an IANA time-zone ID. Confirm the
   response is `202 Accepted` with status `PENDING_DELIVERY`; it must not reveal whether the target
   email already has an account. Confirm a second request for the same inviter and normalized target
   creates neither another invitation nor another outbox message.
6. Read the outgoing invitation through `GET /api/v1/invitations?direction=outgoing`. Confirm
   successful delivery changes its status to `PENDING`. If it becomes `DELIVERY_FAILED`, call
   `POST /api/v1/invitations/{invitationId}/delivery-retries` and confirm the same invitation returns
   to `PENDING_DELIVERY` without changing its original seven-day expiry.
7. Confirm an invitation in `PENDING_DELIVERY` or `DELIVERY_FAILED` cannot be accepted. At the exact
   expiry boundary, confirm delivery, retry, and acceptance fail. After
   delivery changes it to `PENDING`, accept it from the verified target account with
   `POST /api/v1/invitations/{invitationId}/accept`.
8. Using separate test invitations for each case, exercise
   `DELETE /api/v1/invitations/{invitationId}` from every active state. Verify
   concurrent cancel-versus-accept permits only one terminal result, while cancel-versus-retry and
   cancel-versus-delivery-result leave `CANCELLED` final. A provider handoff may already have occurred,
   but the delivered link remains unusable.
9. Submit text with `POST /api/v1/jars/{jarId}/entries`. Confirm the response is empty. If a covered
   operation returns `429`, verify `Retry-After` is present and retry after the indicated delay.
10. Attempt `GET /api/v1/jars/{jarId}/entries` before unlock. Confirm it returns a locked problem
    with no entry content, count, ID, author, or timestamp.
11. In automated tests, advance the injected clock to the exact unlock instant and verify the entry
    page becomes readable without a background transition.

## 6. Reference performance scenario

Before claiming the specification's latency targets, record the environment and workload:

- application instance count and resources;
- PostgreSQL version and resources;
- network placement;
- 100 concurrently active pairs;
- jar-size distribution up to at least 10,000 entries;
- write/read request mix, duration, and warm-up;
- p50, p95, and p99 latency plus error rate.

The stated two-second write and three-second first-page goals apply only to this documented scenario.

## 7. Stop local services

```powershell
docker compose --file .\compose.yaml down
```

Use `docker compose --file .\compose.yaml down -v` only when intentionally discarding local database
data.
