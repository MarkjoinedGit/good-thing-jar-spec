# Frontend Interface Contract

**Feature**: [spec.md](../spec.md) | **Date**: 2026-10-01
**Implementation root**: `C:\workspace\good-thing-jar\good-thing-jar-front-end`

This contract defines UI and client coordination only. Canonical business API:
`C:\workspace\good-thing-jar\good-thing-jar-spec\specs\001-shared-locked-jar\contracts\openapi.yaml`.
The [operation mapping](../api-compatibility.md) applies in full. Do not publish a duplicate
OpenAPI file or introduce endpoints, wire error codes, replay guarantees, email changes, or session rules.

## Routes and User Interface

| Route | Public/private | Contract |
|---|---|---|
| `/register` | Public | Email/password form; accepted registration directs to manual verification, not private app |
| `/verify-email` | Public | Labeled token input and replacement-email form, no token in URL/history |
| `/sign-in` | Public | Generic failure, always-available verification navigation, no saved credentials |
| `/` | Private | Load pair/jar state; show unpaired invitation onboarding or current jar only after backend results |
| `/invitations` | Private | Incoming/outgoing lists with all seven statuses, explicit refresh, email/zone creation, pasted-ID selection |
| `/invitations/:invitationId` | Private | Resolve from incoming/outgoing list; review zone/status/expiry before explicit eligible action; no individual GET |
| `/jars` | Private | History with currentness separate from lock; explicit next-jar creation only from confirmed opened current jar |
| `/jars/:jarId` | Private | Detail, locked composer or authorized entry reader, pending proposal and role-neutral controls |

Protected routing is a UX guard, not authorization. Loading, request error, confirmed empty, denied,
and unknown outcomes are distinct rendered states. Route IDs alone grant no access; no entry text,
email, token, password, invitation form value, or proposal form value goes in route state/URLs.
Sign-out is a button invoking the session flow, not a navigation-only action.

Every route supports desktop/mobile, keyboard and touch. Forms use labels, linked errors, suitable
autocomplete semantics without application persistence, visible focus, and announced outcomes.
No hover-only action. Responsive controls retain the same functions; long content wraps at 320 CSS
pixels/200% zoom. Vietnamese status examples: “Đang tải”, “Chưa có lời nhắn”, “Đang gửi”,
“Đã lưu vào hũ.”; errors contain safe actions and no raw request/response payload.

## API Access Boundary

Feature API modules are the only feature-level callers of shared fetch. Presentation calls
application hooks/flows and never performs fetch or token handling. DTO models follow
[data-model.md](../data-model.md), including optional/nullable fields and empty 204.

- Relative `/api/v1` URLs, prefix retained by proxy; no cross-origin deployment assumption.
- JSON request/response bodies as contracted; `Authorization: Bearer <opaque access token>` only
  on protected operations. No bearer on public registration/verification/resend/sign-in/refresh.
- No credentials in cookies; `credentials: 'omit'`, `cache: 'no-store'`. No secret-bearing URL parameters.
- Protected requests capture generation/revision and consume AbortSignal. Guard all success,
  error, token installation, cache invalidation and UI callbacks against obsolete generations.
- Public auth/verification operations also have attempt IDs so delayed sign-in cannot replace
  a newer session. No request/response-body logging, telemetry or unredacted thrown error objects.
- Map safe allowlisted statuses/codes/violations to Vietnamese; display correlation ID if helpful.
  Arbitrary problem `detail` or request values are never echoed. Resource 404 uses the same safe
  unavailable message for absent/unauthorized resources.
- Query and mutation retries are disabled. Protected work cannot queue for automatic reconnect.
  Manual retries observe cooldown and recovery state. No undocumented idempotency headers.

## Renewal Coordination Contract (FR-005 / US5)

1. One coordinator owns all callers sharing this tab's in-memory token pair. Its state is scoped
   to generation and credential revision. There is no token exchange across tabs.
2. A current-revision protected 401 pauses new work with that revision and initiates or joins the
   single refresh promise. Install the promise synchronously before network dispatch; React rerenders,
   concurrent hooks and Strict Mode must not create a second request. A stale 401 after a successful
   revision change uses that completed renewal outcome instead of starting another refresh.
3. Refresh sends the currently owned refresh token once via the existing public refresh endpoint.
   Each waiter observes the same result, including rejection/cooldown/uncertainty. There is no retry
   library policy around refresh. An expired browser-clock estimate is not a terminal server result.

| Refresh outcome | State and permitted recovery |
|---|---|
| 200 with valid DTO | Atomically install both returned tokens/expiries, increment revision, resume active session only if generation still matches |
| 401 | Generic authentication rejection; immediately terminate local session. No invented expiry/reuse/revocation distinction |
| 429 | Shared Retry-After cooldown; retain draft in memory, do not assume revocation, pause rejected protected work. After wait, explicit user recovery joins one new attempt |
| Network/timeout/ambiguous server or malformed success response | Uncertain: rotation may have committed. Block blind reuse and protected recovery; offer explicit sign-in, clearing private state when chosen. Do not claim remote revocation or continuation |

4. After successful renewal, a denied GET may replay once with the installed revision and original
   cursor/resource. That request has a recovery budget of one: a further 401 yields paused recovery
   guidance and an explicit sign-in/manual-recovery option, never a refresh loop or automatic logout
   based solely on that 401. It cannot turn failed reads into empty results.
5. POST and DELETE operations are never automatically replayed. Explain that the original access
   attempt was rejected and action must be explicit after recovery. Preserve drafts unless actual
   termination occurred. Unknown-outcome mutations follow the separate uncertainty guidance below.
6. Do not retry a refresh with the previous token after a lost response. Backend rotation, previous
   hash reuse detection and committed revocation remain unchanged, including denial of rotated access
   after reuse. No grace period or same-token success guarantee is assumed.
7. Cleanup invalidates a pending refresh by generation. Its later response cannot reinstall tokens,
   resolve private callbacks as successful, or populate a new session's cache.

## Teardown Contract (FR-006 / US5)

On logout or local termination, synchronously invalidate generation/block new work first, detach
the private UI, clear credentials/drafts/form secrets/private caches (both query and mutation),
and start cancellation/abort of old requests. Do not wait for cancellation or network revocation
before clearing. Every eventual completion must still check generation; abort is best effort.
Use a new QueryClient and fresh feature state for subsequent sign-in. Back navigation, bfcache
restoration and delayed handlers cannot restore previous-session state; revalidate guarded shell
state on pageshow and clear references on termination/page lifecycle disposal.

Logout captures the current bearer solely for one best-effort DELETE request before discarding local
credentials. This request is not aborted by the generic private-work cleanup, has no retry/renewal,
and can update only a nonprivate confirmation outcome: 204 confirms revocation, anything else says
local logout complete with remote revocation unconfirmed. An expired bearer may be rejected before
the backend logout controller runs. Never claim remote revocation from local clearing alone.

## Writing Contract (FR-015–FR-018)

Submit exact nonblank plain text to the selected current locked jar. Validate the original text's
1–5000 UTF-16 code units using JavaScript `text.length`, matching Java `String.length()`.
Whitespace-only rejection follows the backend's existing `@NotBlank`, whose classification remains
authoritative. Preserve valid text exactly, including surrounding spaces, line breaks and combining
sequences; never trim or normalize for counting or submission. Freeze draft while pending and
suppress repeated activation. Store no optimistic entry, receipt, count or stored metadata.

Boundary fixtures cover nonblank BMP text at 1/5000 units, a supplementary emoji at 2 units,
2500 supplementary emoji at 5000 units, those emoji plus one BMP character at 5001, and 2501
supplementary emoji at 5002. Empty, over-limit, spaces-only and tabs/line-breaks-only input is
rejected; exact mixed nonblank text with surrounding whitespace is accepted within the limit.
A browser whitespace check must not impose a broader rule than backend `@NotBlank`.

| Outcome | UI / draft behavior |
|---|---|
| 204 | Generic confirmation only; clear draft and mutation variables/references, no body parse |
| Explicit 400 | Safe validation guidance; retain exact draft |
| Explicit 429 | Wait per Retry-After; operation was not performed; retain draft, manual eligible retry |
| 401 | Coordinate session renewal; do not replay mutation; retain draft if session continues |
| 409 | Reconcile backend jar/currentness; read-only/stale guidance; never submit to another jar automatically |
| 404 | Safe unavailable resource; no existence/membership inference |
| Timeout/disconnect/ambiguous 5xx or response | Outcome unknown; original draft retained; no automatic resend. Manual resubmission requires acknowledgement that the original may be saved and another copy may result |

Draft switches retain per-jar memory or ask explicit discard. Unlocked/stale jar draft can remain
unsent in memory until explicit discard or session termination; it is never a read of stored content.
Reload/tab-close loss is explained. No locked-entry list, count, preview, search or receipt check.

## Reading / Jar / Proposal Contract

- Detail confirms backend UNLOCKED before entry query; each read still needs backend permission.
  Browser clock/countdown never infers unlock. Countdown expiry may request one backend revalidation.
- 423 clears any entry-page view/cache for that jar and shows locked/unavailable state; no stored
  metadata. 401 recovery and 404 denial also hide unauthorized content while resolved.
- Cursor pages use `limit=100`, opaque cursor ≤512 and explicit load-more. `hasMore=true` must have
  nonempty nextCursor. Empty final items with `hasMore=false` are a real empty/end result, not error.
  Keep ordering, retry the same failed cursor, detect repeated cursors/duplicate IDs as protocol
  errors, and offer explicit restart on invalid cursor. Bounded rendered pages remain navigable.
- Render notes as text with wrapping and preserved whitespace; never `dangerouslySetInnerHTML`.
  Authors and proposers use returned account IDs, no inferred “you”, email identity, or profile names.
- Confirmed next-jar creation/acceptance invalidates pair/jar/invitation reads as appropriate. A
  conflict refreshes actual state without compensating POST, duplicate email, or another jar.
- Proposal input converts fixed-zone local time to explicit instant. Gap requires correction;
  overlap requires valid offset selection. Pending time is separate from effective time. Neutral
  controls explain only other partner may approve/reject and proposer may cancel; backend checks
  roles. No optimistic effective-date change and no action to relock an opened jar.
- Delivered PENDING invitations may be accepted only after review; retry only eligible failed
  delivery and cancel outgoing active states. Backend status/expiry/races remain decisive.

## Rate Limits, Privacy and Test Seams

Parse contracted Retry-After whole seconds ≥1; hold a monotonic, operation-family cooldown across
callers, suppress early requests and expose Vietnamese wait feedback. If header is unexpectedly
missing/invalid, present safe service guidance and suppress automatic retry; invent no success
guarantee or backend error code. A 429 is distinct from an uncertain write.

No localStorage/sessionStorage/IndexedDB persistence, Query persistence, service-worker caching,
offline queued writes, analytics or sensitive diagnostics. API response caching is disabled.
Private test traces, screenshots, videos, storageState, network bodies and tokens are not persisted.
Use synthetic test data and safe result summaries. MSW handlers live in tests; mock browser startup
is explicitly opted in for a dedicated test build, with production builds and real-backend tests
rejecting mock mode. Mock worker intercepts only for tests and stores no private cache.

Real backend fixtures run in Node test processes only against an allowlisted disposable database;
they never create browser-accessible test endpoints or change production Clock behavior. Exact
backend boundary and browser handling are documented separately, not conflated with browser time.
