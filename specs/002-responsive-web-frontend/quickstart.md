# Frontend Implementation and Verification Quickstart

**Feature**: [spec.md](spec.md) | **Date**: 2026-10-01

Commands below define the planned implementation scripts. The frontend repository is currently
empty: they become runnable after bootstrap, and none is reported as passed by this planning run.
Planning artifacts stay in `C:\workspace\good-thing-jar\good-thing-jar-spec`.

## Prerequisites and Bootstrap

- Node.js 24 LTS with npm; record exact versions and verify chosen stable dependency engines/peers.
- Java 21 and backend Maven wrapper; Docker with PostgreSQL/Mailpit and Testcontainers support.
- Browser access to frontend 5173 (development), 4173 (preview), backend 8080 and Mailpit 8025.
- Isolated synthetic-account E2E database `good_thing_jar_e2e`, never production/personal data.

Bootstrap React/strict TypeScript/Vite in
`C:\workspace\good-thing-jar\good-thing-jar-front-end`, install the selected compatible stack,
ESLint/types and test tooling, and commit frontend `package.json` plus npm `package-lock.json`.
Use npm only and record `packageManager`; regenerate lockfile intentionally with dependency changes.
Set TypeScript `strict`, `isolatedModules`, `noUncheckedIndexedAccess`, and `exactOptionalPropertyTypes`.
Keep real/mock configuration separate; no credentials in `VITE_*` environment variables.

## Planned Frontend Scripts

| Script | Command / behavior |
|---|---|
| `dev` | `vite --host 127.0.0.1`; real backend by default |
| `typecheck` | `tsc -b --pretty false`; includes app/test configurations |
| `lint` | `eslint . --max-warnings 0` |
| `build` | `tsc -b && vite build`; production, mock mode rejected |
| `preview` | `vite preview --host 127.0.0.1 --port 4173 --strictPort` |
| `test` | `vitest run`; RTL, MSW and logic/adapter tests |
| `test:e2e` | `playwright test --project=mock-desktop --project=mock-mobile`; dedicated test-only MSW build |
| `test:e2e:backend` | `playwright test --project=backend-desktop --project=backend-mobile`; actual backend and production-build preview |

Build a dedicated mock test bundle in a separate output directory via a test-only entry/config;
MSW cannot enter production or real-backend runs. Test script definitions include that setup before
Playwright. No production service worker or private caching is added.

From the frontend directory after bootstrap:

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-front-end
node --version
npm --version
npm ci
npm exec playwright install chromium webkit
npm run typecheck
npm run lint
npm run build
npm test
npm run test:e2e
```

Install Edge/Chrome channels if unavailable when checking actual desktop targets. Playwright mobile
viewport/emulation is layout evidence, not a claim of having tested actual Android/iOS. Record
actual Chrome Android/Safari iOS device-browser checks separately for the required browser matrix.

## Backend Environment

Run backend commands from its own root. Supply existing `GTJ_DB_PASSWORD`,
`GTJ_THROTTLE_SCOPE_KEY` and `GTJ_VERIFICATION_DELIVERY_KEY` securely in the backend process
environment; keys must meet existing backend configuration. Do not print, commit or include them
in frontend bundles/test reports. Compose password and application password must agree.

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-backend
docker compose up -d postgres mailpit
docker compose exec -T postgres psql -U good_thing_jar -d postgres -c "CREATE DATABASE good_thing_jar_e2e"
$env:GTJ_DB_URL = 'jdbc:postgresql://localhost:5432/good_thing_jar_e2e'
$env:GTJ_DB_USERNAME = 'good_thing_jar'
.\mvnw.cmd spring-boot:run
```

CREATE DATABASE is first-use setup; reuse an already created E2E database only with fixture
reset/isolation. Flyway applies existing migrations; no schema changes are requested. This process
blocks while running; keep it in its own terminal. Existing health is
`http://localhost:8080/api/v1/actuator/health`; Mailpit inbox is `http://localhost:8025`.
No email link, authenticated identity endpoint, or test clock endpoint is assumed.

## Proxy and Local Browser Startup

In Vite config use both `server.proxy` and local `preview.proxy`:

```typescript
const proxy = {
  '/api/v1': { target: 'http://localhost:8080', changeOrigin: true },
};
// server: { proxy }, preview: { proxy }
// No rewrite: the backend servlet context already includes /api/v1.
```

From frontend root run `npm run dev` for interactive prototype testing. Requests use relative
`/api/v1`; secrets never use frontend environment config. On a phone, expose the Vite host on the
local network intentionally and record origin/network setup; backend remains behind the proxy.
Production deployment requires static frontend hosting with SPA route fallback and same-origin
`/api/v1` reverse proxy to backend. Local Vite preview is verification tooling, not a hosting plan.

## Real-Backend Suite

Create Node-only fixture helpers in frontend `tests/e2e/fixtures`. They use `pg` against the
allowlisted `good_thing_jar_e2e` database only, checking host/database and a dedicated fixture opt-in
before any writes. Connection configuration is server-side test environment, never browser code.
Seed only run-tagged synthetic data, respect existing FK constraints/migrations, and remove only
that run's fixtures after tests. Do not truncate/drop unverified databases or reset personal data.

The test runner takes fixture connection details via private environment variables such as
`GTJ_E2E_DB_URL`; backend uses its existing JDBC URL. Mailpit helpers inspect synthetic emails in
memory, extract actual current tokens/IDs, and never log email bodies/secrets. Respect actual
outbox delivery status and rate limits. Tests use a dedicated backend process; no mock responses,
public test endpoints or production clock changes.

From frontend root, with actual backend and fixture configuration ready:

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-front-end
$env:GTJ_E2E_FIXTURES_ENABLED = 'true'
npm run test:e2e:backend
```

This script builds the production frontend, starts preview on 4173, and forbids MSW/interception of
positive compatibility responses. Fault-injection browser tests are kept separate and labeled.
Disable trace, video, screenshot, storageState export and request/response body output for private
flows in every suite. Redact assertion errors containing payloads. Browser memory and fixture
credentials must not become test artifacts. Use safe test IDs/statuses/aggregate assertions only.

Run existing backend evidence from its root in a separate terminal:

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-backend
.\mvnw.cmd '-Dtest=AuthenticationIntegrationTest,JarLockBoundaryIntegrationTest' test
```

This verifies existing rotation/reuse/revocation and exact trusted-clock unlock behavior. It does
not substitute for browser integration. For exact boundary, record three distinct layers:

1. Backend test denies at unlock minus 1 ns and permits exactly unlock using existing test Clock.
2. Both browser layouts consume locked/unlocked boundary responses through MSW and do not grant
   access on browser-clock changes/countdown expiry alone.
3. Actual backend browsers deny a seeded future locked jar and read once server time is at/after
   its boundary. Fixtures may set synthetic effective times before server startup/action or use a
   short future instant and wait/poll for server confirmation. Browser clocks do not advance server
   time. Do not claim a nanosecond-controlled Playwright server clock.

## Acceptance and Failure Matrix

Execute every acceptance scenario in spec US1–US7 at desktop 1280×800 and mobile 390×844;
also inspect 320 CSS pixels, 200% zoom, keyboard-only operation, focus, labels and announcements.

| Area / requirements | Automated behavior evidence | Real-backend evidence |
|---|---|---|
| Account US1, FR-001–004 | Validation limits, generic acknowledgements, stale attempts, manual token UI | Register/actual email token, superseded/expired verification, resend absent/unverified/verified, generic sign-in including existing 403, logout |
| Invitations US2, FR-008–012 | Every status, optional counterpart, pasted accessible ID, safe unavailable state | Actual invitation-ID email, exact verified recipient/zone review, cancel/accept races, duplicate target, seven-day expiry, wrong recipient, pairing invalidation |
| Delivery retry | Pending differs from delivered; failed retry controls | On isolated environment stop Mailpit SMTP, wait for DELIVERY_FAILED, restore Mailpit, explicit eligible retry keeps ID/expiry. Isolate this disruptive test from other runs |
| Locked writing US3, FR-015–018 | Original UTF-16 length 0/1/5000/5001; supplementary emoji 2/5000/5001/5002-unit fixtures; spaces-only and tabs/line-breaks-only rejection; exact surrounding whitespace/combining text; pending duplicates, 400/429, uncertain response, no replay | Same fixtures against backend String.length()/@NotBlank; real 204 empty/generic confirmation; exact valid payload without trimming/normalization; locked privacy and stale/unlocked denial |
| Reading US4, FR-013–014/019–020 | Clock drift/countdown, 423, malformed/repeated cursor, interrupted-page retry, text-only rendering | Two partners/unrelated account, real locked denial, 10,000 synthetic entries traversed exactly once per partner/layout, author/time/ordering, empty/end and no separate unlock operation |
| Sessions US5, FR-005–007 | Concurrent 401 single refresh; stale 401; refresh 200/401/429/uncertain; no mutation replay; delayed query/refresh/mutation cleanup; offline no queue | Expire access via fixture; actual rotating refresh; revoke session; explicit old-token reuse test in isolated session verifies rotated access then denied; logout including unconfirmed network outcome |
| Privacy/errors FR-007/026–027 | No raw diagnostics/persistence, safe unknown/unauthorized messages, monotonic Retry-After suppression | Browser storage/cache inspection, same absent/unauthorized resource response, actual 429 header/wait/manual action; thresholds remain backend configuration |
| History US6, FR-021 | Currentness distinct from lock; no automatic conflict retry/draft migration | Both partners create next jar concurrently, single current result, old jar remains readable |
| Proposals US7, FR-022–025 | Pending/effective separation, explicit DST overlaps/gaps, neutral roles | Other-partner approval/rejection, proposer cancellation, wrong-role denial, future expiry/one-pending/conflict/opened read-only rules |
| Responsive/accessibility FR-028–030 | RTL role/label assertions and both Playwright layouts, 320px/zoom/focus/touch | Full desktop prototype and mobile journeys against backend, actual target browser versions recorded |

Network-uncertainty tests may deliberately interrupt traffic; they must label fault injection and
must not pretend an intercepted response verifies the backend contract. Real refresh 429 can be
induced using the existing authentication throttle in the isolated environment; do not add error
codes or retry assurances. Coordination also has deterministic MSW race tests to avoid timing flakes.

Entry boundary fixture definitions: one supplementary emoji such as U+1F600 occupies two UTF-16
code units, so 2500 repetitions are valid at 5000; appending one BMP character gives 5001, and
2501 repetitions give 5002, both over limit. Whitespace-only fixtures use spaces and tabs/line
breaks rejected by existing `@NotBlank`; Unicode whitespace edge cases follow backend classification.
Count original input including surrounding spaces/line breaks. Assert valid exact draft/request
text and subsequent authorized readback, including combining sequences, without normalization
or trimming and without exposing stored content while locked or in persisted test artifacts.

## Evidence and Completion

Create `C:\workspace\good-thing-jar\good-thing-jar-spec\specs\002-responsive-web-frontend\verification.md`
during implementation. Record commit IDs, exact Node/npm/dependency/browser versions, backend
configuration with secret values omitted, database/environment identity, commands and exit statuses,
scenario IDs per layout/browser, exact-boundary evidence layers, and any unresolved discrepancy.
Do not record private tokens, account credentials, email bodies or entry content.

Completion requires passing type checking, linting, production build, relevant automated tests,
SC-001–SC-008 and real-backend affected-flow verification. Mocks alone cannot complete the feature.
SC-009's ten-person usability trial remains deferred non-blocking product validation.

Next workflow command: `/speckit.tasks`. Planning does not create `tasks.md` or execute implementation.
