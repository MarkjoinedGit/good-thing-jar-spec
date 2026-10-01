# Research: Responsive Good Thing Jar Web Frontend

**Date**: 2026-10-01 | **Feature**: [spec.md](spec.md)

Research resolves the implementation choices for the requested stack. Backend findings are source
inspection, not a claim that runtime verification has passed. No backend or canonical OpenAPI changes
are selected. Exact installed package versions will be recorded during implementation bootstrap.

## 1. Browser application and dependencies

**Decision**: React SPA with strict TypeScript, Vite, declarative React Router, TanStack Query,
CSS Modules, npm, Vitest, React Testing Library, MSW, and Playwright. Use Node 24 LTS. At bootstrap,
verify stable package engines and peer ranges together, pin the actual selected dependency versions,
record Node/npm versions and `packageManager`, generate and commit `package-lock.json` in the frontend
repository. Subsequent installs use `npm ci`. Planning creates neither packages nor a fabricated lockfile.

**Rationale**: Routing and server state have separate responsibilities. Vite supports CSS Modules and
transpiles TypeScript; an explicit type-check gate is required. Node 24 satisfies the documented
Vite and Vitest runtime requirements. See [Vite](https://vite.dev/guide/),
[Vite TypeScript/CSS support](https://vite.dev/guide/features.html),
[TypeScript strict mode](https://www.typescriptlang.org/tsconfig/strict.html),
[React Router](https://reactrouter.com/start/declarative/installation),
[Node release schedule](https://github.com/nodejs/Release),
[Vitest prerequisites](https://vitest.dev/guide/migration/), and
[npm ci](https://docs.npmjs.com/cli/v11/commands/npm-ci/).

**Alternatives considered**: Node 22 satisfying current engine requirements is viable, but 24 LTS
is the selected baseline. Router framework mode, SSR, Redux, CSS-in-JS, and a general repository
abstraction add unnecessary machinery for these seven user journeys.

## 2. Feature structure and API types

**Decision**: Features `auth`, `invitations`, and `jars` each separate presentation, application,
and API access. Shared code contains the fetch boundary, session coordinator, safe error mapping,
date conversion, and small accessible UI primitives. Use manually maintained narrow DTO types
matching the canonical OpenAPI schemas and runtime guards for required response fields, enums,
optional/null values, and cursor consistency. Contract-focused tests cover these adapters.

**Rationale**: There are 21 bounded business operations. Small explicit adapters are reviewable
without generated SDK layers or duplicate published OpenAPI files. Treat response JSON as unknown;
compile-time typing alone cannot validate server input. The backend contract stays authoritative.

**Alternatives considered**: Generated SDKs, a second OpenAPI copy, and a runtime schema framework
are unnecessary initially. Revisiting generation requires a concrete maintenance need.

## 3. Session renewal and termination

**Decision**: One memory-only session per independently signed-in tab, with a session generation,
credential revision, and one shared refresh promise. Requests carry their initiating generation and
revision internally. Protected 401 joins renewal, not immediate termination. Older-revision 401
cannot start another renewal. Refresh outcomes are 200, generic 401, explicit 429, or uncertainty;
none adds a backend error distinction. See [interface contract](contracts/frontend-interface.md).

**Rationale**: Backend `SessionService.refresh` locks the current/previous token row, rotates both
tokens, and commits revocation on previous-token reuse. Access lifetime is 15 minutes; refresh
retains its original 30-day deadline across rotation. A lost refresh response may follow committed
rotation, so blind reuse is unsafe. Existing `AuthenticationIntegrationTest` verifies rotation,
reuse rejection, subsequent rotated-access denial, and logout revocation. No cookies, JWT identity
claims, or authoritative signed-in account ID are supplied.

**Alternatives considered**: Refresh per failing request, periodic retries, token sharing across
tabs, persistent sign-in, and parsing opaque tokens conflict with existing guarantees or scope.

## 4. Query policy and private state

**Decision**: Set query and mutation `retry: false`; disable automatic focus/reconnect refetch and
interval polling initially. Use explicit refresh controls and targeted invalidation after confirmed
mutations. Configure protected operations to fail promptly offline rather than queue for reconnect
(`networkMode: 'always'` with retry disabled). Consume AbortSignal and guard every completion by
generation. Use a fresh private QueryClient per sign-in; terminate by invalidating generation,
blocking requests, aborting/cancelling, clearing query/mutation caches and drafts/tokens, and resetting
private UI. No persistence plugin, service worker, private HTTP cache, Query Devtools, or analytics.

**Rationale**: Default Query reads retry and stale queries may refetch automatically; unmount alone
does not cancel or prevent later caching. `fetch` uses `cache: 'no-store'` for API requests. Memory
cleanup must include mutation variables and callbacks, not just query data. See
[Query defaults](https://github.com/TanStack/query/blob/main/docs/framework/react/guides/important-defaults.md),
[cancellation](https://github.com/TanStack/query/blob/main/docs/framework/react/guides/query-cancellation.md),
and [QueryClient](https://github.com/TanStack/query/blob/main/packages/query-core/src/queryClient.ts).

**Alternatives considered**: Default automatic retries, persisted Query caches, offline mutations,
and cache clearing without generation guards leave hidden replay or stale-data risks.

## 5. Browser/backend integration and compatibility findings

**Decision**: Relative `/api/v1` requests; Vite development proxy targets backend origin
`http://localhost:8080` and retains the prefix. Configure the same proxy for local production-build
preview tests. Production hosting needs equivalent same-origin routing and SPA fallback excluding
`/api/v1`; Vite proxy is local tooling, not production hosting. No new CORS/API/email changes.

**Rationale**: The backend servlet context is `/api/v1`; no CORS configuration was found. Existing
Compose provides PostgreSQL 18 and Mailpit (SMTP 1025, inbox 8025). Verification emails contain
tokens; invitation emails contain IDs. Lists, not public invitation lookup, resolve those IDs.
See [Vite proxy](https://vite.dev/config/server-options.html#server-proxy).

Source audit found two existing discrepancies to preserve and verify:

- `JarDtos.CreateEntryRequest` has `@NotBlank` and `@Size(max=5000)`; service length is Java
  `String.length()` (UTF-16 units). OpenAPI describes 1–5000 without this whitespace rule. Use JS
  `text.length` on original text, reject whitespace-only according to existing backend `@NotBlank`,
  and preserve valid surrounding spaces/line breaks exactly. A1 clarification now makes those
  frontend acceptance rules explicit. Test 1/5000-unit BMP text, a supplementary emoji (2 units),
  2500 such emoji (5000 units), those emoji plus one BMP character (5001), 2501 emoji (5002), and
  empty/spaces-only/tabs-and-line-breaks-only inputs against the backend. Do not normalize or trim,
  or substitute a broader browser whitespace classification for backend validation.
- Sign-in implementation returns `403/email_not_verified` for correct unverified credentials;
  OpenAPI lists 200/401/429. Tolerate the existing 403 through the same generic sign-in failure and
  always-available verification route, without adding account-state claims or changing either source.

**Alternatives considered**: Removing the proxy prefix, requiring CORS, inventing `/me`, email-link
routes, idempotency keys, write receipts, or new public test endpoints would change the baseline.

## 6. Time-zone conversion

**Decision**: Add `@js-temporal/polyfill` as a focused runtime dependency for proposal conversion.
Display returned instants through `Intl.DateTimeFormat('vi-VN', { timeZone: jar.timeZone, ... })`.
Convert local date/time in the fixed jar zone using rejection of overflow and DST ambiguity. For
overlaps, show both valid offset choices; for gaps, require a corrected local time. Serialize the
chosen instant as an ISO UTC string. Backend decides future eligibility; browser time only assists
display and never grants authorization.

**Rationale**: Explicit ambiguity/offset handling avoids silently shifting proposal instants. See
[Temporal ZonedDateTime conversion](https://tc39.es/proposal-temporal/docs/zoneddatetime.html) and
[polyfill project](https://github.com/js-temporal/temporal-polyfill). Verify polyfill/browser time-zone
behavior against backend Java zones in integration tests; server validation remains final.

**Alternatives considered**: Browser-local `Date` parsing loses jar-zone semantics. Handwritten
offset-search logic is more complicated and easier to get wrong than one bounded utility.

## 7. Testing and exact unlock evidence

**Decision**: Vitest/RTL/MSW for behavior and controlled faults; Playwright for desktop/mobile
layouts, plus a separate suite using actual backend traffic and actual Mailpit emails. Use synthetic
accounts in a disposable E2E database. A test-process-only `pg` dev dependency seeds fixtures and
changes synthetic jar/session records after verifying the database name/host are the designated
E2E database. Database credentials never enter browser bundles.

Exact boundary evidence combines existing backend `JarLockBoundaryIntegrationTest` (trusted
MutableClock at unlock minus 1 ns and exactly unlock), browser/MSW states at both instants on both
layouts, and real-backend browser denials/reads on each side of a controlled seeded boundary.
Production `Clock.systemUTC()` cannot be advanced by Playwright. No public clock-control endpoint
or production clock change is planned. Record these evidence layers accurately.

**Rationale**: Mock scenarios reproduce transport races; actual HTTP, database, and email verification
establish compatibility. Nanosecond boundary correctness belongs to the existing backend clock test,
while both browser layouts prove correct handling of its responses. Disable trace/video/screenshot
and persisted storage-state artifacts for private flows; reports contain scenario IDs, statuses,
counts, versions, and safe correlation IDs, not secrets or entry bodies. See
[RTL](https://testing-library.com/docs/react-testing-library/intro/),
[MSW](https://mswjs.io/docs/quick-start),
[Playwright configuration](https://playwright.dev/docs/test-configuration), and
[Playwright recording options](https://playwright.dev/docs/test-use-options).

**Alternatives considered**: Mocks alone fail the completion gate. A remotely adjustable backend
clock requires extra backend test infrastructure and is unnecessary for this layered proof.

## Resolution

All architecture choices are resolved. Dependency patches and measured runtime results are
implementation outputs, not unresolved feature decisions. SC-009 is deferred non-blocking validation.
