# Feature Specification: Responsive Good Thing Jar Web Frontend

**Feature Branch**: `002-responsive-web-frontend`
**Created**: 2026-10-01
**Status**: Draft — ready for planning
**Input**: A responsive React web frontend with Vietnamese text and a warm, minimal style for
account verification, partner invitations, private locked writing, unlocked reading, jar history,
and mutually approved unlock-time changes using the existing backend.
**Affected Applications**: Frontend; existing backend is an integration dependency.

This separate frontend specification preserves [backend business rules](../001-shared-locked-jar/spec.md)
and [existing API capabilities](../001-shared-locked-jar/contracts/openapi.yaml). Neither source is
overwritten or redefined. The [compatibility review](api-compatibility.md) maps supported operations,
documents limitations, and lists deferred API/email changes separately. No new API or email-content
change is in scope.

## Clarifications

### Review 2026-10-01 - Session renewal

- A protected-operation 401 denies that access attempt but does not establish that session renewal
  is impossible. Continuation follows the existing refresh operation's outcome, without new error codes.
- Backend renewal rotates tokens and previous-refresh-token reuse revokes the session. Concurrent
  callers sharing a session must coordinate renewal. Verified constraints and a candidate approach
  are preserved in [session-renewal review notes](session-renewal-notes.md); implementation
  decisions are deferred to `/speckit.plan`.

### Session 2026-10-01

- Q: Should the SC-009 usability trial block prototype completion? → A: Defer it as follow-up
  product validation; it does not block prototype completion. Required automated checks and
  desktop/mobile verification against the real backend still apply.

### Review 2026-10-01 - Entry validation (A1 resolved)

- Entry length is 1–5000 UTF-16 code units, consistent with Java `String.length()` and JavaScript
  `text.length`. Whitespace-only input is rejected according to the backend's existing `@NotBlank`
  validation. Preserve valid text exactly, including surrounding spaces and line breaks; never trim
  or normalize it. Non-BMP characters such as a single supplementary emoji occupy two code units.
  This clarifies the frontend documents without changing backend behavior or canonical OpenAPI.

## User Scenarios & Testing *(mandatory)*

Every acceptance scenario below MUST pass in a desktop browser layout (1280 × 800) and a mobile
browser layout (390 × 844). Responsive checks also cover 320 CSS pixels, 200% zoom, and desktop
keyboard-only operation. Use verified partners, an unrelated account, backend-controlled time,
and successful/failed responses. Browser clock changes MUST NOT alter backend time.

### User Story 1 - Register, verify, and sign in (Priority: P1)

A person registers, enters the token from their verification email, requests a replacement when
necessary, and signs in to their accessible invitations or shared jar.

**Why this priority**: Verified authentication enables all private journeys.

**Independent Test**: Register a new account, verify using the actual emailed token, request a
replacement, sign in, and sign out from both layouts.

**Acceptance Scenarios**:

1. **Given** a signed-out person, **When** valid registration details are accepted, **Then**
   Vietnamese guidance explains verification is required and offers a labeled token-entry form;
   registration alone is not presented as verification or sign-in.
2. **Given** an email containing a token rather than a clickable link, **When** the person pastes
   it and the backend confirms verification, **Then** they can sign in without an email-link route.
3. **Given** an invalid, expired, or superseded token, **When** verification fails, **Then** a safe
   error offers replacement verification without exposing the token in messages or the browser URL.
4. **Given** absent, unverified, and already verified addresses, **When** replacement requests are
   accepted, **Then** each shows the same generic acknowledgement, such as “Nếu phù hợp, email
   xác minh sẽ được gửi. Vui lòng kiểm tra hộp thư.” No account-state claim or delivery guarantee appears.
5. **Given** a replacement has been issued, **When** an older token is submitted, **Then** it cannot
   verify the account; only the newest unexpired token works and invitation expiry is unchanged.
6. **Given** valid verified credentials, **When** sign-in succeeds, **Then** accessible pair and
   invitation state determines the next view. Invalid/unverified credentials receive a generic
   failure without disclosing account existence.
7. **Given** either layout, **When** invalid email, password, or token input is submitted, **Then**
   labeled errors are reachable by keyboard or touch, entered secrets are not echoed, and all
   account actions remain usable without horizontal page scrolling.

---

### User Story 2 - Invite a partner and accept a reviewed invitation (Priority: P1)

A verified unpaired person invites an email with an explicitly selected time zone, inspects
delivery status, cancels active invitations, and retries failed delivery. The recipient verifies
the invited email, reviews an incoming invitation's time zone, and explicitly accepts it.

**Why this priority**: Pairing creates the private two-person relationship and initial jar.

**Independent Test**: Use two verified accounts and an invitation email containing an ID. Exercise
creation, delivery, retry, cancellation, and acceptance on both layouts; test a wrong recipient.

**Acceptance Scenarios**:

1. **Given** an eligible unpaired person, **When** an email and valid time zone are submitted,
   **Then** acceptance shows “Đang gửi”, original expiry, and no recipient account-state claim;
   queued delivery is not represented as successful delivery.
2. **Given** outgoing invitations, **When** status is refreshed, **Then** waiting for delivery,
   delivered, delivery-failed, accepted, cancelled, expired, and invalidated have distinct Vietnamese
   labels. Loading or a failed refresh never appears as an empty list.
3. **Given** an unexpired delivery failure, **When** its inviter retries, **Then** the same invitation
   returns to waiting for delivery without another invitation or an extended deadline. A duplicate
   creation attempt instead gives a safe conflict and does not queue another email.
4. **Given** any active outgoing invitation, **When** cancellation succeeds, **Then** it remains
   terminal despite late delivery. If acceptance wins a race, refreshed state shows the actual pair
   rather than falsely claiming cancellation succeeded.
5. **Given** an email containing an invitation ID, **When** the recipient signs in with the verified
   invited email, **Then** they can select a matching incoming invitation or paste the ID to select
   it from their accessible list, review its time zone, and explicitly accept. The ID alone grants
   no access and does not bypass review.
6. **Given** a delivered eligible invitation, **When** the verified recipient accepts after review,
   **Then** backend confirmation leads to the two-member pair and current locked jar; other
   invitations invalidated by pairing can no longer create a pair.
7. **Given** pending delivery, delivery failure, expiry, cancellation, acceptance, invalidation,
   or a wrong recipient, **When** acceptance is attempted, **Then** no successful pairing is shown;
   stale eligible-looking state is reconciled with the backend.
8. **Given** either layout, **When** an invitation is reviewed or acted upon, **Then** permitted
   counterpart information, time zone, expiry, status, and controls remain readable and reachable
   without hover-only interactions or horizontal page scrolling.

---

### User Story 3 - View the locked jar and collect private notes (Priority: P1)

Either partner views the current jar's zone and effective unlock time, then submits a positive
note while the backend considers it locked. Confirmation acknowledges storage without redisplaying it.

**Why this priority**: Private collection before reveal is the central everyday value.

**Independent Test**: Submit boundary-length text from both partners to a locked jar; inspect its
views and confirmations. Repeat with validation, throttling, interruption, and unlock during writing.

**Acceptance Scenarios**:

1. **Given** a paired authenticated person, **When** the current jar loads, **Then** backend lock
   status, fixed time zone, and effective date/time are clear. An optional approximate countdown
   is informational; loading or error cannot imply unlock.
2. **Given** a current locked jar and nonblank text at valid boundaries, **When** either partner
   submits text of 1 or 5000 UTF-16 code units and receives confirmed success, **Then** the draft
   clears and only generic confirmation such as “Đã lưu vào hũ.” appears, without stored text, ID,
   author, time, count, preview, or recent-entry list. Coverage includes one BMP character, 5000
   BMP characters, one supplementary emoji (2 units), and 2500 supplementary emoji (5000 units).
3. **Given** empty text, text of 5001 UTF-16 code units, or whitespace-only input rejected by the
   backend's existing `@NotBlank` validation, **When** submission is attempted, **Then** validation
   explains the applicable length/nonblank rule and no entry is accepted. Coverage includes 2500
   supplementary emoji plus one BMP character (5001 units), 2501 supplementary emoji (5002 units),
   spaces-only input, and tabs/line-breaks-only input. Nonblank valid text preserves Vietnamese
   accents, surrounding spaces, and line breaks exactly in the draft and submitted payload; length
   includes those spaces and line breaks, with no trimming, normalization, or rewriting.
4. **Given** an outstanding submission, **When** submit is activated repeatedly, **Then** only one
   creation request is outstanding, pending feedback is clear, and no optimistic saved-entry view appears.
5. **Given** explicit recoverable validation or throttling rejection, **When** the composer is shown,
   **Then** the exact draft remains in the active session; manual retry after the stated delay is
   available without a daily entry quota.
6. **Given** an interrupted write with unknown outcome, **When** recovery guidance appears,
   **Then** the original draft remains, no storage result is claimed, and no automatic resend occurs.
   Explicit manual resubmission warns the original may have been saved and another copy may result;
   the frontend does not read locked entries to determine the outcome.
7. **Given** the jar unlocks or ceases to be current during submission, **When** the backend rejects
   the write, **Then** refreshed state removes writing actions and explains read-only/stale status,
   without automatically submitting the draft to another jar.
8. **Given** either layout, **When** a long draft is composed, validated, or submitted, **Then** its
   label, unsaved UTF-16-code-unit length feedback, pending state, and confirmation remain usable with
   touch or keyboard; draft length is never displayed as a stored-entry count.

---

### User Story 4 - Read together after backend-confirmed unlock (Priority: P1)

Either partner reads the opened jar through successive pages with author and creation time.
The jar stays read-only.

**Why this priority**: Reveal completes the collection cycle while preserving its privacy.

**Independent Test**: With a multi-page jar, deny reads immediately before backend unlock, allow
them at the exact boundary, and read every note exactly once on both layouts. Change browser time
independently and test an unrelated account.

**Acceptance Scenarios**:

1. **Given** backend time immediately before unlock, **When** browser time is advanced or a countdown
   reaches zero, **Then** no entry is revealed or unlocked status inferred. Backend denial keeps
   the jar locked without stored content, counts, or metadata.
2. **Given** backend time exactly at or after unlock, **When** a partner refreshes and reads,
   **Then** backend confirmation permits reading without a separate “open jar” mutation or schedule.
3. **Given** multiple entry pages, **When** successive pages are loaded, **Then** each accepted entry
   is reachable in backend order exactly once, with unchanged plain text, returned author account
   attribution, and creation date/time.
4. **Given** another-page loading fails, **When** that read is retried, **Then** loaded authorized
   notes remain ordered, none are skipped or duplicated, and no false empty/end state appears.
5. **Given** an empty opened jar or final page, **When** its confirmed response appears, **Then** a
   meaningful empty/end state replaces further-page controls.
6. **Given** either layout, **When** opened entries are viewed, **Then** long Vietnamese text, author,
   time, and more-page controls reflow without horizontal page scrolling; writing/date changes are absent.
7. **Given** an unauthorized or nonexistent jar reference, **When** a direct visit is attempted,
   **Then** the same safe unavailable message appears without existence or membership disclosure.

---

### User Story 5 - Recover safely from session and service problems (Priority: P1)

A person receives understandable recovery guidance while private data remains in their active
session. Sign-out or session termination clears private views and drafts before another account uses them.

**Why this priority**: Session and failure handling protect every private journey.

**Independent Test**: Load private data and a draft, revoke/end the session, introduce delayed
responses, and sign in as another person. Exercise read recovery, failed writes, and rate limits
on both layouts with actual backend revocation and controlled failures.

**Acceptance Scenarios**:

1. **Given** an active private session, **When** sign-out succeeds, **Then** backend revocation is
   confirmed and tokens, drafts, entries, pair/invitation data, and private query caches are cleared
   before the signed-out view appears.
2. **Given** a network failure prevents confirming revocation, **When** sign-out is selected,
   **Then** local state still clears immediately; remote revocation is not falsely claimed and
   re-entry requires sign-in.
3. **Given** a protected operation returns 401 and renewal credentials are available, **When** the
   existing refresh operation returns 200, **Then** the session continues with the returned rotated
   credentials and its draft is retained; the protected 401 alone does not sign the person out.
   No mutation is automatically replayed, including an entry submission that triggered recovery.
4. **Given** a request pending at termination, **When** it later completes or another account signs
   in, **Then** it cannot restore old data/drafts. Back/forward navigation cannot reopen ended-session content.
5. **Given** a recoverable read failure, **When** recovery appears, **Then** retry is available and
   loading/empty/error remain distinct. Failed or uncertain mutations are not silently replayed
   during recovery or renewal.
6. **Given** HTTP 429 with Retry-After, **When** an operation is rejected, **Then** Vietnamese guidance
   gives the wait, suppresses early attempts, and retains a rejected entry draft; later manual retry
   does not disclose account existence or impose a daily limit.
7. **Given** either layout and keyboard use, **When** loading, empty, error, or pending states occur,
   **Then** meaning and recovery are accessibly announced without secret or draft/entry diagnostics.
8. **Given** renewal is attempted, **When** the refresh endpoint returns 401, **Then** local session
   termination clears private state/drafts and prompts sign-in. Guidance remains generic and does
   not distinguish invalid, expired, revoked, or reused tokens.
9. **Given** renewal is attempted, **When** the refresh endpoint returns 429 with Retry-After,
   **Then** renewal waits for that delay and private drafts remain in the current session. The person
   is not told their session is revoked; protected work does not continue on rejected access, and
   any later renewal attempt is coordinated rather than promised to succeed.
10. **Given** a renewal response is lost or a network failure leaves its outcome uncertain, **When**
    recovery guidance appears, **Then** the application neither claims session continuation nor
    backend revocation and does not blindly resubmit the refresh token. It offers sign-in recovery;
    choosing that recovery terminates local state and clears drafts without replaying mutations.
11. **Given** multiple protected operations in the same session concurrently return 401, **When**
    renewal is needed, **Then** they share one renewal attempt and outcome instead of independently
    using the same refresh token. Successful renewal supplies the same new credentials to those
    callers; a delayed 401 from replaced credentials does not start a duplicate renewal.
12. **Given** the previous refresh token is deliberately reused after rotation in a backend test,
    **When** the backend rejects renewal and subsequently denies the rotated access token,
    **Then** the frontend uses generic termination/cleanup and never works around reuse detection
    or claims a refresh-token grace period.

---

### User Story 6 - Revisit previous jars and start the next collection (Priority: P2)

Partners revisit opened collections and create one new locked jar after the current jar opens.

**Why this priority**: History and subsequent jars extend the experience across years.

**Independent Test**: Seed previous jars and an opened current jar, visit each, and race next-jar
creation from both partners on desktop and mobile.

**Acceptance Scenarios**:

1. **Given** previous jars, **When** history opens, **Then** sequence and dates distinguish jars,
   currentness is clear, and selection loads that jar's own backend status and authorized entries.
2. **Given** the current jar is backend-confirmed opened, **When** the next is created, **Then**
   confirmation identifies a new current locked jar in the agreed time zone; the old one stays readable/read-only.
3. **Given** simultaneous next-jar requests, **When** one conflicts, **Then** refresh shows the same
   single new jar; the loser neither claims another jar nor repeatedly attempts creation.
4. **Given** a locked current jar, **When** history is viewed or a stale creation action is attempted,
   **Then** next-jar creation is unavailable and backend denial cannot be bypassed.
5. **Given** either layout, **When** moving between history, previous jars, and the current jar,
   **Then** navigation stays accessible and the selected jar clear; drafts are not silently submitted
   to another jar.

---

### User Story 7 - Agree on a different future unlock time (Priority: P2)

Either partner proposes a future instant. Only the other partner may approve/reject; the proposer
may cancel. Both can review the unchanged effective time and separate proposed time.

**Why this priority**: Mutual consent permits meaningful dates without unilateral exposure.

**Independent Test**: Exercise earlier/later future proposals, approval, rejection, cancellation,
wrong-role attempts, and expiry during review on both layouts.

**Acceptance Scenarios**:

1. **Given** a locked jar without a pending proposal, **When** a future instant is proposed,
   **Then** confirmation shows proposer and proposed time separately; effective time, access,
   and its countdown remain unchanged while pending.
2. **Given** an eligible pending proposal, **When** the other partner approves, **Then** only
   backend-confirmed approval changes the effective instant for both partners.
3. **Given** proposer approval/rejection or other-partner cancellation, **When** the backend denies
   it, **Then** no success or date change appears and safe state is refreshed.
4. **Given** a pending proposal, **When** the other partner rejects or the proposer cancels,
   **Then** confirmation removes it, keeps the effective date, and permits a later proposal while locked.
5. **Given** proposed time is no longer future, the jar has opened, or the proposal was resolved,
   **When** a stale action is attempted, **Then** it cannot relock/reschedule; a safe conflict refreshes state.
6. **Given** a pending proposal, **When** another is attempted, **Then** it is not silently replaced
   and the one-pending-proposal rule remains intact.
7. **Given** either layout, **When** entering/reviewing dates, **Then** effective/proposed times,
   fixed zone, proposer identifier, and role rules stay readable. Ambiguous local dates require an
   explicit offset choice or correction; nonexistent daylight-saving times cannot silently shift.

### Edge Cases

- Trusted backend time governs exact unlock. Wrong browser time, sleeping tabs, countdown expiry,
  stale responses, or failed refreshes cannot grant access; refresh failure leaves content unavailable.
- Registration uses 12–128-character passwords; sign-in follows existing credential rules rather
  than imposing the registration minimum. Emails exceed neither the 320-character limit nor valid syntax.
- An invitation may precede registration; verification/resend never extend seven-day expiry.
  Multiple different-target invitations are permitted; duplicate normalized targets, self-invites,
  paired targets, competing acceptance, and terminal states follow safe backend outcomes.
- Cancellation and acceptance may race; late delivery never reactivates cancellation/expiry.
  Missing/inaccessible invitation IDs receive identical safe guidance.
- An uncertain write may have been stored. Its retained composer text is not a stored-entry preview
  or permission to look it up. Preserve recoverable drafts in memory, never persistent storage.
- A draft belongs to one jar/session. Switching jars preserves it in memory for that jar or confirms
  discard; it is never silently discarded or migrated. Reload/tab closure may lose it; termination clears it.
- Interrupted/invalid-cursor pages retry the same read or explicitly restart the collection.
  Cursors are opaque and cannot be mixed across jars/sessions or replaced by guessed page numbers.
- A protected 401 alone does not establish failed renewal. Refresh success, generic authentication
  rejection, explicit throttling, and uncertain transport outcomes are distinct recovery states.
  Failed/uncertain renewal cannot create endless retries, replay a write, or restore an ended session.
- Concurrent callers share renewal. Previous-refresh-token reuse may revoke even the newly rotated
  credentials; a lost success response is not permission to retry the old token or assume a grace period.
- Authors/proposers have returned account IDs, not names. Do not guess caller identity from email
  input or decode opaque tokens; “you/partner” requires authoritative identity evidence.
- Defaults remain January 1 at 00:00:00 of the year after creation in the agreed fixed time zone,
  including rollover/daylight-saving rules. Display the returned instant rather than inventing a default.
- Markup-like notes remain plain text. Vietnamese diacritics, emoji, long strings, zoom, and an
  on-screen keyboard must not corrupt valid text or hide core actions.
- Entry length counts original UTF-16 code units, not grapheme clusters or Unicode code points.
  A supplementary emoji counts as two units; combining marks and surrounding spaces/line breaks
  also contribute to length. Backend `@NotBlank` classification is authoritative for whitespace-only
  rejection; do not replace it with trimmed text or assume a browser whitespace check is identical.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The frontend MUST provide Vietnamese registration, manual token verification,
  replacement verification, sign-in, and sign-out. Private flows require backend-authorized sessions;
  invitation acceptance requires verification of the invited email.
- **FR-002**: Validation MUST follow existing email/password/token/time-zone/entry limits with
  accessible field guidance, without replacing backend validation or silently rewriting text.
- **FR-003**: Registration, sign-in, resend, invitations, and throttling MUST preserve generic
  responses without adding account-existence or verification-state disclosure or delivery guarantees.
- **FR-004**: Verification MUST accept the emailed token explicitly and offer safe replacement
  guidance for invalid/expired/superseded tokens; no clickable email links are assumed.
- **FR-005**: A protected-operation 401 MUST NOT alone establish that renewal is impossible. With
  available renewal credentials, session recovery MUST use the existing refresh endpoint's outcome:
  200 adopts rotated credentials and continues the session; 401 terminates local state with generic
  authentication guidance; 429 honors Retry-After without assuming revocation; network uncertainty
  confirms neither continuation nor revocation and MUST NOT trigger blind refresh-token reuse.
  Offer sign-in recovery when continuation cannot be established, clearing state on local termination.
  Missing renewal credentials require sign-in. Local expiry times MUST NOT authorize access, and
  recovery MUST NOT automatically replay mutations. Callers sharing a session MUST coordinate
  renewal, share its outcome, and avoid independent same-token or stale-response renewal attempts;
  the mechanism belongs in the plan. Preserve existing rotation, reuse detection, and revocation.
- **FR-006**: Sign-out MUST attempt backend revocation and immediately clear tokens, private state,
  drafts, and private query caches even when remote revocation is uncertain. Expired/revoked termination
  MUST do the same. Delayed responses/browser history MUST NOT restore ended-session data.
- **FR-007**: Tokens, passwords, verification secrets, entry content, and sensitive data MUST NOT
  appear in logs, analytics, URLs, error details, or persistent application/browser caches. Private
  state/drafts MUST be nonpersistent; entries MUST render as plain text, never executable markup.
- **FR-008**: Eligible verified unpaired users MUST invite emails with an explicitly selected valid
  IANA time zone, including before registration. Recipient account/pairing state MUST NOT be disclosed.
- **FR-009**: Provide separate incoming/outgoing lists with permitted counterpart information,
  zone, expiry, and all seven statuses. Pending delivery MUST differ from successful delivery/failure.
- **FR-010**: Inviters MUST cancel any active invitation or retry eligible failed delivery using the
  same invitation/expiry. Duplicates, terminal states, seven-day expiry, and races follow backend decisions.
- **FR-011**: Recipients MUST select an accessible incoming invitation, optionally by entering its
  emailed ID, review its time zone, and explicitly accept. Only delivered eligible invitations to
  their verified email may succeed; an ID alone grants neither access nor disclosure.
- **FR-012**: Confirmed acceptance MUST lead to the backend-created two-member pair and initial jar.
  Membership, other-invitation invalidation, and exactly-one-pair rules remain backend-authoritative;
  refresh after acceptance or conflicts.
- **FR-013**: Both partners MUST view current/previous jars with sequence, fixed zone, effective
  instant, and backend lock status. Currentness differs from lock status: the current jar may be open.
- **FR-014**: Browser clocks/countdowns/routes/state MUST be display-only for access. Unlocked status
  and protected reading require backend confirmation. Countdown expiry, pending proposals, and
  failed requests MUST NOT infer permission or reveal entries.
- **FR-015**: Either partner MUST submit exact plain text of 1–5000 UTF-16 code units only to the
  authorized current locked jar, consistent with Java `String.length()` and JavaScript `text.length`.
  Whitespace-only input MUST be rejected according to the backend's existing `@NotBlank` validation.
  The frontend MUST preserve valid text exactly, including surrounding spaces and line breaks,
  and MUST NOT trim or normalize it for counting or submission. Unsaved draft text/length MAY be
  shown; stored counts and daily quotas MUST NOT. Backend validation remains authoritative.
- **FR-016**: Confirmed submission MUST clear its draft and show only generic confirmation. Locked
  views MUST NOT show stored text, IDs, authors/times, counts, previews, lists, search, exports, or
  other stored-entry reads, including the author's own entries.
- **FR-017**: Show pending submission and prevent repeated outstanding creation requests. Explicit
  recoverable rejection MUST retain the exact draft in its session. Success/termination clear it;
  navigation MUST NOT silently submit it to another jar.
- **FR-018**: Uncertain creation MUST NOT automatically retry or claim known storage success/failure.
  Retain original draft with uncertainty guidance; explicit resubmission MUST warn of duplicates.
  No assumed entry-idempotency capability is in scope.
- **FR-019**: After confirmed unlock, both partners MUST reach every entry exactly once in stable
  cursor pages with exact text, returned author attribution, and creation time. More-page, loading,
  retry, empty, and end states MUST NOT skip, duplicate, or reorder entries.
- **FR-020**: Opened jars MUST stay readable/read-only under session authorization, without writes
  or date changes; previous opened collections remain accessible.
- **FR-021**: Either partner MUST start the next jar after current unlock with the agreed zone and
  backend default date. Concurrent conflicts MUST refresh to one shared current jar, not fabricate another.
- **FR-022**: Either partner MUST propose a future instant while locked; show proposer/proposed time
  separately. Effective time, countdown, and access remain unchanged while pending; only one proposal is pending.
- **FR-023**: Only the other partner may successfully approve/reject; only the proposer may cancel.
  Explain roles and report success only after confirmation. Without authoritative caller identity,
  use neutral action wording and backend-validated outcomes rather than guessed permissions.
- **FR-024**: Only eligible confirmed approval changes the effective date. Rejection, cancellation,
  expiry, stale actions, and opened jars MUST NOT relock/reschedule; conflicts give safe refreshed state.
- **FR-025**: Dates MUST show the jar's fixed zone. Proposal input MUST resolve an unambiguous instant;
  invalid/ambiguous local times require correction/disambiguation. Future eligibility is backend-authoritative.
- **FR-026**: HTTP 429 MUST communicate a Vietnamese wait and honor Retry-After before another
  operation attempt. Explicitly throttled writes were not accepted and can be manually retried later;
  uncertain outcomes are distinct. No existence disclosure or daily quota is allowed.
- **FR-027**: Session, connectivity, validation, conflict, locked-read, and resource errors MUST have
  actionable Vietnamese states. Unauthorized/nonexistent protected resources share safe unavailable
  messaging without existence/membership disclosure. Failed reads MUST NOT look like empty results.
- **FR-028**: All flows MUST work in responsive desktop/mobile browsers. The initial prototype MUST
  be fully usable/testable in desktop; mobile preserves every core action without native apps or
  requiring a desktop fallback.
- **FR-029**: Forms MUST have accessible labels, keyboard-reachable controls, visible focus, and
  meaningful accessible loading/empty/error/validation/pending feedback. Touch MUST NOT depend on hover.
- **FR-030**: Use Vietnamese text and a warm, minimal style; status MUST NOT depend on color alone.
  Core content/controls MUST reflow at 320 CSS pixels and work at 200% zoom without horizontal page scrolling.
- **FR-031**: Use only existing capabilities. Required API/email changes MUST be documented separately
  and explicitly admitted to scope before implementation; deferred compatibility proposals are not requirements.

### Key Entities *(include if feature involves data)*

- **Account/session**: Registration/verification and renewable private access; the contract lacks
  authoritative signed-in account identity.
- **Invitation**: Direction, ID, permitted counterpart, zone, status, and seven-day expiry; only
  delivered eligible invitations can form a pair.
- **Pair**: Exactly two distinct verified accounts sharing jars and an agreed zone.
- **Jar**: Sequence/currentness, fixed zone, effective instant, backend lock status, optional proposal.
- **Draft**: Session-only composer text for one jar, not a retrievable entry; its text may remain
  after uncertain creation without a known storage result.
- **Entry/page**: Post-unlock text, author account ID, creation time, opaque continuation cursor.
- **Unlock proposal**: Proposed instant/proposer and pending, approved, rejected, cancelled, or
  expired state; eligible approval alone changes the effective instant.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every required journey passes at 1280 × 800 desktop and 390 × 844 mobile; all core
  controls stay reachable at 320 CSS pixels and 200% zoom. The complete initial account-to-reveal
  prototype journey is demonstrable entirely in a desktop browser.
- **SC-002**: All account/invitation/jar/proposal journeys work with keyboard alone, visible focus,
  labels, and meaningful state announcements. User-facing text is Vietnamese except identifiers,
  email addresses, and time-zone names.
- **SC-003**: Every locked-read test reveals zero stored content/counts/metadata, including immediately
  before unlock, changed browser clocks, stale state, and countdown expiry. Unauthorized and
  nonexistent resources are indistinguishable in every corresponding test.
- **SC-004**: Both partners retrieve all 10,000 entries in an opened test jar exactly once with
  unchanged text, correct author ID/time, and no skips/duplicates after interrupted-page retry.
  Reads work at exact backend unlock without a separate opening action.
- **SC-005**: All submission tests accept nonblank text of 1–5000 UTF-16 code units and reject empty,
  over-limit, and whitespace-only input according to existing backend validation. Boundary coverage
  includes 1/5000-unit BMP text, a supplementary emoji (2 units), 2500 supplementary emoji (5000
  units), those emoji plus one BMP character (5001 units), and spaces-only/tabs-and-line-breaks-only
  input. Valid surrounding spaces, line breaks, accents, and combining sequences remain exact,
  without trimming or normalization. Successful drafts clear with generic confirmation, recoverably
  rejected drafts remain exact, and uncertain outcomes cause zero automatic resubmissions.
- **SC-006**: Every termination test clears private data/drafts before another account uses the app;
  delayed responses/history never restore them. Inspection finds no tokens/entry text in
  diagnostics, analytics, or persistent caches.
  All renewal tests distinguish protected 401 from refresh 200/401/429 and transport uncertainty,
  preserve drafts until termination, perform no automatic mutation replay, and make exactly one
  shared refresh attempt for concurrent renewal triggers without reusing rotated-out tokens.
- **SC-007**: All invitation tests preserve delivery, expiry, verified-email acceptance, and terminal
  states; all proposal-role tests preserve mutual consent and unchanged pending effective time.
  Concurrent next-jar attempts show exactly one new jar.
- **SC-008**: Every rate-limit test communicates the wait, suppresses early attempts, and permits
  eligible manual retries without a daily quota. No failure adds account/resource-existence disclosure.
- **SC-009 (Follow-up product validation; non-blocking)**: After the prototype is available, run a
  documented trial with at least 10 invitees split between desktop/mobile, targeting at least 90%
  registering, entering the emailed token, and accepting a time-zone-reviewed invitation unaided
  within ten minutes after both required emails are available. Measure delivery wait separately.
  This deferred trial and its target do not block prototype completion.

## Assumptions

- React is the user's delivery constraint. Structure, dependencies, test tools, and commands belong
  in planning; this document specifies behavior rather than implementation design.
- Specs/plans/contracts stay in `good-thing-jar-spec`; frontend implementation stays in sibling
  `good-thing-jar-front-end`. Backend implementation remains in `good-thing-jar-backend` as a dependency.
- Emails currently contain a token or invitation ID. Manual token entry and authenticated
  invitation-list selection are baseline flows. Email links, public lookup, and password reset
  are not assumed capabilities.
- Authors/proposers use returned account IDs or stable distinct labels with their ID available.
  Friendly names and “you/partner” labels are not required because caller identity/profiles are absent.
  Proposal controls use neutral role explanations and backend outcomes; role-based hiding is deferred
  unless an authoritative identity/capability contract is separately approved.
- Sessions/drafts are not persisted across reload, closure, or restart; reload may require sign-in.
  Persistent sign-in/offline private browsing are excluded; supported renewal continues the current
  in-memory session.
- The planned browser matrix covers stable Chrome/Edge desktop, Chrome Android, and Safari iOS,
  recording actual versions during verification. Native Android/iOS apps, installation/offline
  features, and push notifications are excluded.
- Entry editing/deletion, partner replacement/dissolution, social feeds, image/file uploads, reminders,
  account deletion, and pair time-zone changes are excluded. No daily quota, locked stored-entry
  summaries, or relocking of opened jars is introduced.
- Backend/email test environments and documented browser access configuration are dependencies;
  deployment concerns do not authorize new APIs/email changes.
- Completion follows constitution v1.1.0: type checking, linting, production build, relevant automated
  authentication/privacy/unlock tests, and real-backend verification MUST pass. Plans/tasks record
  commands, environments, scenarios, and results; mocks alone cannot establish completion.
- SC-001 through SC-008 and the constitution completion gates apply to prototype acceptance.
  SC-009 remains a separate follow-up product-validation target and MUST NOT be treated as a
  prerequisite for prototype acceptance or completion.
