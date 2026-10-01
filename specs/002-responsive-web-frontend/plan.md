# Implementation Plan: Responsive Good Thing Jar Web Frontend

**Branch**: `002-responsive-web-frontend` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)
**Input**: `C:\workspace\good-thing-jar\good-thing-jar-spec\specs\002-responsive-web-frontend\spec.md`
and the user's requested React/TypeScript/Vite stack.
**Status**: Planning complete; implementation and runtime verification pending.

## Summary

Build a responsive Vietnamese React SPA for verified sign-in, reviewed partner invitations,
locked private writing, cursor-based unlocked reading, jar history, and mutual unlock proposals.
Use the existing Spring Boot API unchanged. Separate presentation, flows, and API access within
three feature modules. Coordinate rotating-token renewal centrally; keep private state in memory;
respect backend lock/role decisions and uncertain-write outcomes. Desktop and mobile verification
against the real backend is required alongside automated checks.

## Technical Context

**Affected Applications**: Frontend; existing backend is an integration/test dependency, with no production changes planned.
**Language/Version**: Strict TypeScript, React; Node.js 24 LTS and npm. Exact compatible stable package versions recorded at bootstrap and locked in package-lock.json.
**Primary Dependencies**: Vite, React Router, TanStack Query, CSS Modules, @js-temporal/polyfill; ESLint, Vitest, React Testing Library, MSW, Playwright, pg (test-process-only fixture tooling).
**Storage**: N/A
Private tokens, drafts, query data, and mutation state use session-scoped memory only. PostgreSQL
belongs to the existing backend; disposable E2E fixture access is confined to test tooling.
**Testing**: Vitest/RTL/MSW behavior tests, Playwright mocked and real-backend browser suites; existing Maven backend authentication and exact-unlock integration tests.
**Target Platform**: Stable Chrome/Edge desktop, Chrome Android, Safari iOS; 1280×800 and 390×844 acceptance layouts, 320 CSS-pixel reflow, 200% zoom, keyboard operation.
**Project Type**: Browser frontend in a separate sibling repository.
**Performance Goals**: All core controls usable at required layouts; cursor traversal reaches all 10,000 fixture entries exactly once without skipped/duplicated pages. No speculative latency SLO added.
**Constraints**: Existing opaque-token API and email token/ID flows; no persistent private state, mutation replay, new endpoints, backend session changes, stored locked-entry metadata, or browser-authorized unlock.
**Scale/Scope**: Seven stories; P1 US1–US5, P2 US6–US7; two-person pairs, nonblank entries of 1–5000 UTF-16 code units per existing backend validation, cursor limit up to 100. SC-001–SC-008 acceptance gates; SC-009 deferred.

## Constitution Check

Gates assessed before research and re-assessed after design against constitution v1.1.0.
“Pass” evaluates the design; it does not assert implementation completion.

| Gate | Before research | After design / evidence |
|---|---|---|
| Correctness and API compatibility | Pass: preserve backend specification and OpenAPI | Pass: existing operations only; contract differences recorded in research and compatibility review, not changed |
| Repository boundaries | Pass: three sibling roots | Pass: documentation here, all frontend code/tooling/lockfile in `C:\workspace\good-thing-jar\good-thing-jar-front-end`, backend commands in its root |
| Backend controller/business/persistence separation | N/A: no backend production change | N/A: existing backend remains the integration authority |
| Backend persistence/transactions/migrations | N/A: no production schema change | N/A: isolated test fixtures only; retain backend transactional race protection |
| Security and observability | Pass: memory-only private data | Pass: no-store fetch, session generation/revision, coordinated refresh, cache/draft cleanup, safe errors and reports |
| Frontend architecture | Pass: requested simple feature structure | Pass: UI → application flows → feature API → shared transport; no UI fetch calls |
| Backend authority | Pass: browser clocks display-only | Pass: backend detail plus protected page responses gate reads; role checks and pending effective time unchanged |
| Responsive accessible interaction | Pass: required browser journeys | Pass: route/UI contract and test matrix cover labels, focus, keyboard, touch, reflow, loading/empty/error/pending |
| Testing and completion | Pass: real backend required | Pass: quickstart lists all gates and layered exact-boundary evidence; results must be recorded during implementation |
| Maintainability | Pass: limited features and shared concerns | Pass: focused Temporal utility for DST correctness, pg only for isolated fixture tooling; no global state framework or generic SDK |

No constitution exceptions or amendment required.

## Project Structure

### Documentation (this feature)

All paths below are under
`C:\workspace\good-thing-jar\good-thing-jar-spec\specs\002-responsive-web-frontend\`:

```text
spec.md
api-compatibility.md
session-renewal-notes.md           # historical review; decisions now in this plan/contracts
plan.md
research.md
data-model.md
quickstart.md
contracts/frontend-interface.md   # UI/client interface, references canonical backend OpenAPI
checklists/requirements.md
tasks.md                          # future /speckit.tasks output; not generated by planning
verification.md                   # future implementation evidence; not a fabricated pass report
```

### Source Code (separate sibling application directories)

Planned frontend structure, rooted at
`C:\workspace\good-thing-jar\good-thing-jar-front-end\`:

```text
package.json
package-lock.json
index.html
vite.config.ts
tsconfig*.json
eslint.config.js
vitest.config.ts
playwright.config.ts
src/
  app/                            # router, providers, shell, global style tokens
  features/
    auth/{presentation,application,api}/
    invitations/{presentation,application,api}/
    jars/{presentation,application,api}/
  shared/
    api/                          # fetch, typed safe outcomes, DTO guards
    session/                      # memory credentials, renewal, teardown
    time/                         # zone display and proposal conversion
    ui/                           # labels, status, focus, buttons when actually shared
  test/                           # Vitest setup and MSW handlers
tests/e2e/
  mocked/                         # MSW browser scenarios
  backend/                        # actual HTTP/email journeys
  fixtures/                       # Node-only pg and Mailpit helpers
```

Co-locate `*.test.ts(x)` with behavior, and `*.module.css` with presentation. Do not create unused
subdirectories or abstractions. Backend implementation remains rooted at
`C:\workspace\good-thing-jar\good-thing-jar-backend\`, with its existing `src/main/java/com/goodthingjar/`
and `src/test/java/com/goodthingjar/` modules. Specifications and contracts stay in the spec root.

**Structure Decision**: Three cohesive features are sufficient. Jar composition, reading, history,
and proposals share jar DTOs and invalidation, so they stay in `jars`; session security is shared
because all private features depend on it. Router handles navigation, Query owns server state,
React state/application stores own ephemeral forms/drafts. No entry content in route state or URLs.

## Design Decisions

### Session and API boundary

Use one session-scoped renewal coordinator, in-memory tokens, generation-tagged requests and
credential revisions. Install the shared promise before sending refresh. Protected 401 joins it;
old-revision 401 after rotation uses the installed result instead of refreshing again. Share
200/401/429/uncertain outcomes, not just successful promises. Refresh is never automatically retried.
At most one safe GET replay may occur after successful renewal; no mutation is replayed, including
DELETE operations. Further 401 during a read recovery surfaces authentication guidance without a
loop. Details and teardown order are in [frontend-interface.md](contracts/frontend-interface.md).

Do not put token responses in Query cache. Entry mutation variables must not retain confirmed
submission text: reset/remove completed mutation state, clear the draft, and retain only generic
confirmation. Every late callback checks generation before UI, cache, token, or draft updates.
Sign-out captures a best-effort revocation request and immediately terminates locally; no silent
refresh for logout and no claim of remote revocation unless 204 was received.

### Server-state and mutations

Private QueryClient per session, query keys scoped by generation and resource ID; all retries,
focus/reconnect refetch, polling, and paused offline mutation resume disabled. Use explicit refresh,
manual read retry, targeted invalidation, and no optimistic authorization/entry-content updates.
Countdown zero may request one guarded jar revalidation; it cannot enable reading. Query enabled
conditions include active session, no blocked recovery, no cooldown, and backend-confirmed jar
unlock for entry-page queries. Denied 423 removes entry pages and returns to locked/unavailable UI.

All commands disable repeated outstanding submission. 400/429 retains draft; uncertain outcomes
retain it with a duplicate-warning confirmation before manual resubmission. Successful 204 clears
only the submitted jar's draft. A 401 can renew the session but leaves mutation resubmission explicit.
409 reconciles pair/jar/invitation state through reads, never repeated creation. Generic 404 and
unknown resource states cannot leak existence or membership.

Entry validation uses original `text.length` (UTF-16 code units), matching Java `String.length()`;
backend `@NotBlank` remains authoritative for whitespace-only input. Valid surrounding spaces and
line breaks count toward the limit and are submitted unchanged. Boundary tests include one and
2500 supplementary emoji (2 and 5000 units), emoji plus BMP text at 5001 units, and whitespace-only
rejection. No trimming or normalization is used for length, submission, or display.

### Dates, reading, and role presentation

Backend fields determine `current`, `lockStatus`, and `effectiveUnlockAt`. Display fixed jar zone
and effective/proposed times separately. Temporal rejects nonexistent local proposal times;
overlaps require one explicit valid offset choice. Local expiry/clock values never terminate a
renewable session or grant access without a backend result. No caller ID is supplied, so author
and proposer labels use returned IDs and proposal actions explain roles neutrally.

Use `useInfiniteQuery` with opaque cursors and `limit=100`; continuation follows `hasMore` plus
returned nonempty cursor, not item count. Retry a failed page with its unchanged cursor. Invalid
cursor offers an explicit restart. Prevent concurrent load-more requests. Keep previously loaded
pages in session memory and use an accessible paged reading viewport over those pages to bound
rendered DOM; visited pages remain selectable by earlier/later controls. Never fabricate backend
page numbers or counts. Guard duplicate IDs/repeated cursors as protocol failures, not silent skips.

### Responsive and accessible UI

Warm neutral backgrounds, restrained accent colors, Vietnamese copy, readable spacing, CSS Module
layouts with mobile-first reflow. Desktop shell and mobile navigation preserve the same routes and
actions. Use semantic forms/buttons, linked field errors, visible focus, status announcements,
pending button labels and explicit disabled explanations. Invitation review is a full routed view
usable without hover. Long text wraps and preserves whitespace as plain text. Confirm draft discard
on jar navigation; retain per-jar drafts in active-session memory otherwise. Display reload-loss
guidance; teardown bypasses draft persistence or prompts.

## Implementation and Verification Sequence

1. Bootstrap frontend in its sibling directory: runtime/dependency compatibility check, strict
   TS config, npm scripts, lockfile, router/providers, CSS tokens and test infrastructure.
2. Implement shared transport/session coordinator and teardown before private features; verify
   renewal outcomes, concurrent/stale 401s, uncertain mutation and late-response cleanup.
3. Implement P1 US1–US5: manual verification and resend, reviewed invitations, locked composer,
   authorized reading, safe recovery. Test both layouts at each complete journey.
4. Implement P2 US6–US7: history/next jar and explicit-offset proposals; preserve backend race
   outcomes and neutral role controls.
5. Run all completion gates and actual backend/email journeys, including fixture-based scale,
   privacy, lock, revocation, delivery-failure/retry, throttling and concurrency checks. Record
   exact commands, versions, environments, scenarios and results in `verification.md`.

This is the sequencing input for `/speckit.tasks`, not generated implementation tasks. Testing
scope and commands are in [quickstart.md](quickstart.md). SC-009 remains a non-blocking follow-up.

## Complexity Tracking

No constitution violations. No additional architectural layers require an exception.
