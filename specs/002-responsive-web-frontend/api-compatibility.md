# Frontend API and Email Compatibility Review

**Feature**: [Responsive Good Thing Jar Web Frontend](spec.md)
**Reviewed**: 2026-10-01
**Sources**: [Backend specification](../001-shared-locked-jar/spec.md),
[OpenAPI 1.0.0](../001-shared-locked-jar/contracts/openapi.yaml), constitution v1.1.0,
and the user's description of current email contents. Session behavior was additionally checked
against backend `SessionService`, `AuthSessionEntity`, `AuthSessionRepository`, the authentication
filter/entry point, and the existing rotation/reuse/logout integration test on 2026-10-01.
Planning additionally inspected entry DTO/service validation, mail gateway/outbox, servlet context,
Compose, production Clock configuration, and existing exact-unlock tests.

This review records current capabilities, not a replacement contract or a promise about an
unverified deployed environment. All paths below use the `/api/v1` prefix. No backend specification,
contract, or email template is changed by this frontend specification.

## Supported Capability Mapping

| Frontend flow | Existing operation | Compatibility and user-visible outcome |
|---|---|---|
| Register (US1) | `POST /auth/registrations` | `202` returns account ID and verification-required state, never the verification secret. Generic `409` does not identify a registered email. |
| Enter emailed token (US1) | `POST /auth/email-verifications` | Body contains `token`; `204` confirms verification. No automatic sign-in or clickable-link assumption. |
| Replacement verification (US1) | `POST /auth/email-verification-resends` | Body contains email; `202` is identical for absent, unverified, and verified addresses. Only newest unexpired token works; `429` respects Retry-After. |
| Sign in (US1) | `POST /auth/sessions` | Opaque access/refresh tokens and expiry instants; `401` has generic sign-in guidance and `429` has wait guidance. Tokens are not decodable identity claims. |
| Continue a renewable session (US5) | `POST /auth/sessions/refresh` | A protected-operation `401` alone does not establish failed renewal. Share one coordinated refresh: `200` adopts rotated credentials; refresh `401` terminates with generic guidance; `429` waits per Retry-After; transport uncertainty neither confirms revocation nor permits blind same-token retry. Never replay mutations. |
| Sign out (US5) | `DELETE /auth/sessions/current` | `204` confirms immediate revocation; `401` already denies access. Clear local private state even if the request fails, without claiming confirmed remote revocation. |
| Find accessible pair (US1–3) | `GET /pairs/current` | Returns exactly two member IDs and time zone. Protected `404` offers neutral onboarding/unavailable guidance; it must not explain resource existence or membership. |
| Invite email/time zone (US2) | `POST /invitations` | `202` reports one `PENDING_DELIVERY` invitation. Duplicate/ineligible `409` is privacy-safe; `429` indicates wait. Creation does not wait for SMTP. |
| Inspect or select invitation ID (US2) | `GET /invitations?direction=incoming` or `outgoing` | Authenticated lists contain IDs, direction, permitted counterpart email, status, zone, and original expiry. Match emailed ID against the incoming list; there is no individual GET endpoint. |
| Cancel outgoing invitation (US2) | `DELETE /invitations/{invitationId}` | `204` confirms cancellation in an active state; `409` reconciles races. Protected `404` never discloses existence. |
| Retry failed delivery (US2) | `POST /invitations/{invitationId}/delivery-retries` | `202` returns the same ID in `PENDING_DELIVERY`; original expiry unchanged. Only eligible `DELIVERY_FAILED` supports retry. |
| Accept reviewed invitation (US2) | `POST /invitations/{invitationId}/accept` | `201` returns pair details. Only verified target of delivered eligible `PENDING` succeeds; then fetch jar state. Wrong recipient uses privacy-safe denial. |
| Current jar and history (US3, US6) | `GET /jars` | Use returned `current`, sequence, zone, effective instant, and lock status. No invented `/jars/current` endpoint and no entry counts in this list. |
| Jar detail and pending proposal (US3, US7) | `GET /jars/{jarId}` | Backend-derived `lockStatus` and `effectiveUnlockAt`; optional `pendingUnlockProposal` may be absent or null. No browser-derived permission. |
| Submit exact draft text (US3) | `POST /jars/{jarId}/entries` | Body contains `text`; successful `204` has no entry body or metadata. `400` validates, `409` rejects read-only/stale jars, `429` rejects without accepting. No documented idempotency key or write receipt. |
| Read successive entry pages (US4) | `GET /jars/{jarId}/entries?cursor=...&limit=...` | `200` returns items, `hasMore`, optional `nextCursor`. Limit 1–100, default 50; cursor is opaque, at most 512 characters. `423` denies locked reads without entry metadata. |
| Next collection (US6) | `POST /jars` | `201` creates the next current jar; `409` means locked current jar or a concurrent winner. Refresh instead of creating again automatically. |
| Propose future instant (US7) | `POST /jars/{jarId}/unlock-proposals` | Body contains `proposedUnlockAt`; `201` returns proposer ID and proposal without changing the effective instant. Backend decides future eligibility and one-pending invariant. |
| Other-partner approval (US7) | `POST /jars/{jarId}/unlock-proposals/{proposalId}/approval` | `200` returns updated jar detail; only eligible backend-approved action changes effective time. Wrong-role/stale actions do not succeed. |
| Other-partner rejection (US7) | `POST /jars/{jarId}/unlock-proposals/{proposalId}/rejection` | `204` confirms rejection; effective instant unchanged. |
| Proposer cancellation (US7) | `DELETE /jars/{jarId}/unlock-proposals/{proposalId}` | `204` confirms proposer cancellation; effective instant unchanged. |

Read errors remain distinct from empty results. Protected `404` is intentionally identical for
absent and unauthorized resources. `Problem` exposes safe `code`, `correlationId`, and optional
field violations; translate safe known outcomes into Vietnamese without echoing arbitrary response
details or sensitive request values. Correlation IDs may support investigation without entry text.

## Existing Rules and Limits

- Access expiry is checked separately from refresh usability in `AuthSessionEntity`. A protected
  `401` therefore cannot distinguish expired access from an unrenewable session. The existing
  refresh outcome determines recovery; no new endpoint or error distinction is required.
- `SessionService.refresh` locks the session row, rotates access and refresh tokens, and retains
  the previous refresh hash. Reuse of that previous token revokes the session and returns generic
  authentication rejection, committed despite the exception. Newly rotated access is then denied.
  There is no same-token retry guarantee or grace period. Coordinate concurrent frontend callers
  rather than altering these rules; see [the renewal review notes](session-renewal-notes.md).
- Refresh throttling runs before token lookup/rotation. Explicit `429` leaves renewal postponed
  without evidence of revocation. A lost response might instead follow a committed rotation:
  never blindly resend that refresh token or classify transport uncertainty as backend rejection.
- Registration: email up to 320 characters, password 12–128. Sign-in accepts passwords 1–128;
  do not apply the registration minimum to sign-in.
- Verification token and refresh token requests: 32–512 characters. Verification expires after
  24 hours; replacements supersede previous tokens and do not extend invitation expiry.
- Invitation: selected valid IANA zone, request length 1–64. Expiry is exactly seven days after
  creation, never after delivery/retry. Active statuses are `PENDING_DELIVERY`, `PENDING`, and
  `DELIVERY_FAILED`; `ACCEPTED`, `CANCELLED`, `EXPIRED`, and `INVALIDATED` are terminal.
- Both partners must review/agree to the initial zone. It remains fixed for jars; defaults use
  January 1 at midnight in that zone of the calendar year after creation.
- Entry: exact nonblank text of 1–5000 UTF-16 code units, matching Java `String.length()` and
  JavaScript `text.length`. Reject whitespace-only input according to existing backend `@NotBlank`.
  Supplementary emoji count as two units each: 2500 occupy 5000; adding one BMP character makes
  5001 and exceeds the limit. Preserve valid Vietnamese text, surrounding spaces and line breaks
  exactly, including for counting; never normalize, trim, or rewrite it.
- Backend time determines lock status: before the effective instant reads are denied; at/after it
  reads are permitted to members and writes/date changes are denied. No explicit unlock operation exists.
- `Retry-After` is required whole seconds, minimum 1, on contracted `429`. The operation was not
  performed; a throttled entry is known not to have been accepted. A timeout/disconnect/ambiguous
  server failure is a separate unknown-outcome case and cannot trigger automatic entry retry.
- There is no daily entry quota. Entry list/detail/count metadata cannot be obtained while locked.
- Pair creation and next-jar races must show the backend's single resulting pair/jar. Never compensate
  for conflicts by guessing membership, creating another jar, or changing dates locally.

## Implementation/Contract Differences Found During Planning

These differences already exist; this feature changes neither backend behavior nor canonical
OpenAPI. Real-backend tests must exercise them and record results. Any contract reconciliation is
separate backend review work, not an implicit frontend dependency.

| Source difference | Existing-compatible frontend handling |
|---|---|
| `JarDtos.CreateEntryRequest` has `@NotBlank` and `@Size(max=5000)`; `EntryCommandService` uses Java UTF-16 `String.length()`. OpenAPI states minLength 1/maxLength 5000 without a whitespace-only rule. | Use JS `text.length` for compatible length guidance. Preserve exact valid text; handle server blank rejection without trimming/normalizing. Verify Vietnamese, combining marks, emoji/non-BMP and boundaries against the real backend; do not promise broader acceptance from OpenAPI alone. |
| `SessionService.create` can return existing `403/email_not_verified`; sign-in OpenAPI enumerates 200/401/429. | Tolerate actual 403 with the same generic sign-in failure and verification route offered to everyone. Do not add an account-state claim or expose raw problem details. |

These are recorded discrepancies, not new endpoints or error distinctions introduced by the client.
The existing missing-caller-identity limitation remains unchanged.
Review A1 now explicitly aligns FR-015, US3 and SC-005 with the implementation's UTF-16/nonblank
rules. The canonical OpenAPI discrepancy remains documented and unchanged; real-backend verification
still needs to execute the stated boundary cases.

## Email Flows Without Changes

### Verification token

1. After registration, show a token-entry screen reachable from the signed-out account navigation.
2. The person reads their current verification email and copies its token into the form.
3. Confirmation comes only from the verification operation; then offer sign-in.
4. Replacement uses the email resend operation and generic acknowledgement. Explain that the newest
   email's unexpired token must be used. No token in a URL, analytics, error, log, or persistent cache.

### Invitation ID

1. The recipient receives an invitation ID by email; they may still need to register and verify the
   exact invited address before signing in.
2. The application loads their authenticated incoming list. They can select a listed invitation or
   paste the emailed ID to select a matching accessible item.
3. Show returned inviter information when present, proposed time zone, status, and expiry before
   explicit acceptance. A missing match yields generic unavailable guidance, never public lookup.
4. Only acceptance confirmed by the existing operation creates the pair. Delivery status/expiry,
   different-address sign-in, and cancellation remain backend-controlled.

These flows require neither email links nor a public invitation-detail endpoint. The user's current
email description takes precedence over historical backend prose mentioning an invitation link.

## Documented Limitations and Deferred Changes

No item below is authorized or required implementation work for this feature. Each must first
receive a separate scope decision and compatible contract/email review if it becomes a requirement.

| Limitation | Existing-compatible baseline | Deferred change and impact |
|---|---|---|
| No authoritative caller account ID in `SessionResponse`; no current-account read capability | Show proposer/author account IDs or stable distinct labels with IDs available. Use neutral proposal controls explaining who may act, and only report success after backend role validation. Do not decode opaque tokens or assume registration ID survives later sign-in. | An additive authenticated identity or action-capability response would enable role-based control hiding and reliable “you/partner” labels. It must preserve existing authorization and generic errors; its exact design is separate. |
| No member display names or identity-to-profile mapping | Attribute entries to returned `authorAccountId`; distinguish members without invented names or inferred email associations. | A separately approved profile mapping could provide friendly names with explicit privacy rules. |
| Emails contain token/ID rather than application URLs | Manual token entry and authenticated incoming-list selection by ID. | Clickable email links require email-content and frontend routing decisions, secret/URL exposure review, and explicit scope admission. They do not permit public protected-resource disclosure. |
| No individual invitation GET | Resolve ID within authenticated incoming list before review/acceptance. | A detail capability is optional; none is needed for the baseline. Any future capability preserves absent/unauthorized equivalence. |
| No entry idempotency or submission-status query | Retain original draft, explain unknown outcome, never retry automatically, warn before explicit resubmission. | Idempotency/receipt support would be a separately specified backend contract change; do not send undocumented keys or deduce a receipt from counts. |
| No automatic push/status stream | Refresh authenticated invitation/jar state through supported reads and offer explicit refresh. Countdown completion only prompts backend revalidation. | Push updates or new subscription interfaces are outside this release. |
| Browser access is a deployment dependency, not described by OpenAPI | Plan same-origin routing or verify the backend's permitted frontend origin, bearer requests, and browser visibility of Retry-After. Record actual configuration during real-backend verification. | Any required CORS/hosting configuration is documented before implementation and does not authorize new business endpoints or changed email content. |

The role-neutral baseline deliberately does not promise that unauthorized proposal controls can be
hidden before a request. It guarantees the existing role rule is enforced and never reports a
forbidden action as successful. An identity addition is necessary only if that stronger presentation
requirement is later admitted to scope.

## Planning and Verification Handoff

Plan frontend work in `C:\workspace\good-thing-jar\good-thing-jar-front-end`, with presentation,
application flows, and API access separated, preferably by feature. Keep plans/tasks/contracts and
this review in `good-thing-jar-spec`. Run app commands from the relevant sibling application root.

The plan must identify safe session-only token/draft handling, termination/query cleanup including
late-request races, browser access configuration, date/zone conversion, and behavior-focused tests.
The [review notes](session-renewal-notes.md) preserve verified renewal constraints and a candidate
coordination approach. During `/speckit.plan`, assess the approach and record implementation
decisions without adding endpoints, authentication error distinctions, retry guarantees, or backend
session changes. These notes were written before planning. Planning ran on 2026-10-01; selected
decisions now live in [plan.md](plan.md) and
[contracts/frontend-interface.md](contracts/frontend-interface.md).
Completion requires type checking, linting, a production build, relevant automated tests, and real
backend verification. Use actual verification/invitation emails, two partners plus an unrelated
account, revoked sessions, before/exact-unlock backend time, invitation races, cursor reads,
wrong-role proposals, and Retry-After. Record environments, commands, and results without secrets.

This is specification-level compatibility review, not evidence those runtime checks have passed.
