# Frontend Data Model

**Feature**: [spec.md](spec.md) | **Date**: 2026-10-01

These are browser DTOs and ephemeral application models, not new database tables or backend
entities. The canonical source is
`C:\workspace\good-thing-jar\good-thing-jar-spec\specs\001-shared-locked-jar\contracts\openapi.yaml`.
All private values live only in memory and disappear at session termination/reload.

## Transport Models

| Model | Fields and relationships | Validation / interpretation |
|---|---|---|
| Registration result | `accountId`, `verificationRequired: true` | Registration is not sign-in; ID is not evidence of the caller in a later session |
| SessionResponse | `accessToken`, `refreshToken`, `accessExpiresAt`, `refreshExpiresAt` | Opaque strings and ISO instants; no caller account ID, no token decoding, no Query cache |
| InvitationSummary | `id`, `direction`, optional `counterpartEmail`, `status`, `timeZone`, `expiresAt`, `createdAt` | Directions INCOMING/OUTGOING; counterpart may be absent. Backend determines status and eligibility |
| PairResponse | `id`, `timeZone`, `memberAccountIds`, `createdAt` | Exactly two distinct IDs; no identity-to-email/profile mapping inferred |
| JarSummary | `id`, `sequenceNumber`, `current`, `timeZone`, `effectiveUnlockAt`, `lockStatus`, `createdAt` | Positive sequence; LOCKED/UNLOCKED returned by backend. Currentness is independent of lock |
| JarDetail | JarSummary fields plus optional/null `pendingUnlockProposal` | Absent and null mean no returned pending proposal; no stored-entry summary/count |
| EntryResponse | `id`, `text`, `authorAccountId`, `createdAt` | Obtained only by authorized unlocked read; render exact plain text and returned author ID |
| EntryPage | `items`, `hasMore`, optional/null `nextCursor` | Continuation cursor is opaque and jar/session-bound; `hasMore=true` requires a usable cursor |
| UnlockProposalResponse | `id`, `proposedUnlockAt`, `proposedByAccountId`, `status`, `createdAt` | PENDING/APPROVED/REJECTED/CANCELLED/EXPIRED; pending proposal never overrides effective time |
| Problem | `type`, `title`, `status`, `code`, `correlationId`, optional `detail`, optional field violations | Map allowlisted status/code/field to Vietnamese; do not echo arbitrary text or request values |

DTO guards reject malformed required fields, impossible enums, invalid instants, or inconsistent
cursor pages with safe service-error UI; they neither manufacture permission nor log raw JSON.
Mutation 204 responses are empty success, never parsed as JSON.

## Input Models

| Input | Contract / existing implementation rule |
|---|---|
| Registration | Email valid and ≤320; password 12–128 |
| Sign-in | Email valid and ≤320; password 1–128, not registration minimum |
| Verify token / refresh | Token 32–512; verification input is manual, not a URL value |
| Resend | Email only; accepted result is generic for account states |
| Create invitation | Email and explicitly reviewed IANA zone, 1–64 characters; aliases/zone eligibility remain server decisions |
| Select emailed invitation | ID must match authenticated incoming list; no public resource lookup |
| Create entry | Nonblank text of 1–5000 UTF-16 code units, measured on original text by JS `text.length` / Java `String.length()`; whitespace-only rejection follows backend `@NotBlank`. Count and preserve valid surrounding spaces/line breaks exactly; never trim/normalize |
| Entry page | `limit=100`, cursor absent for first page, otherwise exact returned cursor ≤512 |
| Create proposal | Local date/time + fixed jar zone + explicit offset where ambiguous → one ISO instant; future eligibility is backend-authoritative |

The blank/non-BMP validation discrepancy is documented in [research.md](research.md) and the
compatibility review. Client hints do not create broader acceptance guarantees than the server.

Entry validation fixtures include nonblank BMP text at 1/5000 units, one supplementary emoji at
2 units, 2500 supplementary emoji at 5000, the latter plus one BMP character at 5001, and 2501
supplementary emoji at 5002. Empty, over-limit, spaces-only and tabs/line-breaks-only inputs are
rejected. Mixed nonblank text with surrounding spaces/line breaks and combining sequences remains
unchanged; measure the original text rather than a trimmed or normalized copy. Backend whitespace
classification is final; a broader browser whitespace definition must not narrow accepted inputs.

## Application Models

### Local session

`LocalSession` contains a monotonically changing `generation`, `credentialRevision`, credential pair,
recovery state, pending refresh promise, request AbortControllers, and private QueryClient reference.
Do not use tokens as query keys. A successful sign-in creates a fresh generation and QueryClient.
No browser-storage, BroadcastChannel token exchange, cookie, or multi-tab credential sharing.

| State | Trigger | Next state / effect |
|---|---|---|
| signedOut | Sign-in 200 | active, revision 1, new generation |
| active | Current-revision protected 401 | renewing; all relevant callers share one promise |
| renewing | Refresh 200, same active generation | active; atomically install both tokens/expiries and increment revision |
| renewing | Refresh 401 | signedOut; terminate and clear everything generically |
| renewing | Refresh 429 | throttled; retain draft/data in memory, block rejected credential's work, shared cooldown |
| throttled | User requests recovery after cooldown | renewing with latest retained credentials; no automatic mutation resend |
| renewing | Transport/ambiguous response | uncertain; no old-token replay, suspend protected work, retain draft until explicit recovery |
| uncertain | User chooses sign-in recovery | signedOut; cleanup then fresh sign-in |
| any private state | Logout, replacement sign-in, missing renewal credentials / confirmed terminal rejection | signedOut; generation invalidated before any delayed result can commit |

Returned expiry values may assist messaging but cannot establish authorization, jar access, or
irrecoverability from browser time. A stale 401 uses the installed revision/shared result; it does
not start an independent refresh. No automatic mutation replay in any transition.

### Draft

`Draft` = `{ generation, jarId, text, revision, submissionState }`. Store per jar within active
session memory; never in Query data, router history, browser storage, diagnostics, or URL.
`submissionState` is editing / submitting / rejected / uncertain. A send captures exact text and
draft revision; freeze editing while outstanding, so confirmed success clears the submitted draft
unambiguously. On error, retain its exact value. Clear on success or termination. Navigation keeps
the draft for its jar or explicitly confirms discard; it cannot migrate it to another jar.

On uncertain outcome, the captured original remains the draft and the UI warns that it may have
been saved. User-confirmed resubmission may create a duplicate; there is no receipt/idempotency model.
After successful submission, mutation cache variables and component references are reset/removed.

### Private query keys and read view

- Pair: `['private', generation, 'pair']`.
- Invitations: `['private', generation, 'invitations', direction]`.
- Jar list: `['private', generation, 'jars']`.
- Detail: `['private', generation, 'jar', jarId]`.
- Entries: `['private', generation, 'entries', jarId, 100]`, page parameters are returned cursors.

An `EntryReadingView` holds visited page selection, not a guessed API page number. Previously
loaded pages remain in memory, with bounded rendered DOM and accessible earlier/later navigation.
Detect repeated cursor or duplicate IDs as a safe protocol error requiring restart; don't silently
drop duplicates, advance failed cursors, or treat errors as empty/end. A locked/unauthorized/session
denial clears the relevant private content and never enables a page query.

### Cooldown and operation result

`Cooldown` stores operation family, Retry-After seconds and monotonic timer deadline. Shared auth
cooldown applies to competing refresh/sign-in attempts in this client; entry/delivery/resend
cooldowns suppress repeated corresponding operations. Wait does not imply revocation or quota.
Local clock changes cannot shorten the current wait. After expiration, action is explicitly manual.

Internal outcome categories: success, known rejection, authentication recovery, throttled,
uncertain transport/server outcome, or discarded stale completion. These are client states, not
new wire error codes or promised server distinctions. Any potentially committed mutation failure
without an explicit rejection is conservatively uncertain.

## Backend-Owned Transitions

- Invitations: PENDING_DELIVERY → PENDING or DELIVERY_FAILED; eligible explicit retry returns
  DELIVERY_FAILED → PENDING_DELIVERY with same ID/expiry. Active states may become terminal;
  terminal states never return active. No browser-expiry transition can confirm server action.
- Pair acceptance creates one pair and initial jar, invalidating competing invitations.
- Jar LOCKED → UNLOCKED is computed by backend trusted time. Pending proposals leave effective
  unlock unchanged; approved eligible proposals replace it. Opened jars cannot relock.
- Next jar after current unlock becomes the single new current jar; previous jars remain readable.
- Proposal roles and expiration are backend-validated. Without caller identity, UI never labels
  inferred ownership or treats neutral controls as permission.

No migrations or backend data-model changes are needed.
