# Feature Specification: Shared Locked Jar

**Feature Branch**: `001-shared-locked-jar`
**Created**: 2026-09-18
**Status**: Draft
**Input**: Build a Good Thing Jar backend for couples: authenticated pairing, unlimited positive
entries in a private jar, a default end-of-year or mutually customized unlock date, and access
to all entries only after unlocking. Notifications and reminders are optional.

## Clarifications

### Session 2026-09-18

- Q: Must the pre-unlock lock also prevent service operators from reading stored note text? → A: No;
  the lock must block both partners and other app users. Privileged operational access may exist
  only under strict authorization and audit.
- Q: Can a partner be invited before registering? → A: Yes; send the invitation by email, and
  require the invitee to register and verify that email before accepting.
- Q: Can partners add notes to a jar after it unlocks? → A: No; the opened jar stays read-only,
  and the pair starts a separate locked jar for a new writing period. Opening follows the effective
  unlock instant and trusted system time, without a separate state transition.
- Q: Whose approval is required to change a locked jar's unlock date? → A: Both partners must
  approve every change, whether the date moves earlier or later.
- Q: How is the time zone for the default year-end unlock chosen? → A: The inviter must select a
  time zone, and the invitee must see and accept it before pairing. Future jars use that agreed
  pair time zone.

### Session 2026-09-21

- Q: How is a retryable invitation email failure exposed when delivery is asynchronous? → A: The
  creation request returns an accepted response with pending-delivery status after the invitation
  and its delivery request are reliably recorded together. Delivery later changes the status to
  pending or delivery-failed; the inviter observes that status and retries the same invitation
  without the original request waiting for email delivery.
- Q: When does an invitation's seven-day acceptance window begin? → A: It begins when the
  invitation is created. Delivery and retries do not reset or extend its expiration time; an expired
  invitation cannot be delivered, retried, or accepted, so the inviter must create a new one.
- Q: May one inviter have multiple active invitations for the same target email? → A: No. The
  inviter may have at most one unexpired invitation for the same normalized target email while it
  is `PENDING_DELIVERY`, `PENDING`, or `DELIVERY_FAILED`; duplicate creation is rejected without a
  new invitation or email, and failed delivery uses the retry operation. A new invitation is allowed
  after the prior one is accepted, canceled, expired, or invalidated if the inviter remains eligible.
- Q: In which states may an invitation be canceled? → A: The inviter may cancel it while
  `PENDING_DELIVERY`, `PENDING`, or `DELIVERY_FAILED`. `CANCELLED` is terminal and mutually exclusive
  with `ACCEPTED`. Once cancellation completes, no acceptance, retry, queued delivery, or later
  delivery result may reactivate the invitation. Email already handed off cannot be recalled, but
  its invitation link remains unusable.
- Q: What happens when an email-verification token expires? → A: Tokens expire 24 hours after
  issuance. An unverified account may request a replacement through a resend operation with the same
  generic accepted response whether the email is absent, unverified, or already verified. A
  replacement supersedes every prior unsuperseded token, including expired tokens, only the newest
  unexpired token works, and resending does not change invitation expiry.
- Q: How must the backend throttle requests while preserving unlimited daily entries? → A: It must use
  configurable, operation-specific short-window throttles for authentication attempts, verification
  resend, invitation creation and delivery retry, and entry writes. A throttled request receives a
  retryable response with an indicated delay. Throttling cannot impose a daily entry quota,
  permanently reduce entry capacity, or disclose whether an account, email, invitation, membership,
  or protected resource exists.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Share and open a private jar (Priority: P1)

Two people create accounts and form a private pair. Both can write positive notes, memories, or
messages freely while the jar is locked. Neither can read stored entries until the year-end unlock
instant; then both can read the full collection and begin another jar.

**Why this priority**: This complete pairing, writing, and reveal cycle is the minimum useful
experience for a couple.

**Independent Test**: Register and verify two accounts, pair them, submit many entries from both
partners, confirm that all early reads are denied, then reach the unlock instant and retrieve every
entry. Check a third account is denied throughout.

**Acceptance Scenarios**:

1. **Given** an unpaired verified account holder, **when** they invite a partner by email with a
   selected time zone and that partner registers, verifies the invited email, reviews the time
   zone, and accepts, **then** the pair has exactly two members and a private locked jar.
2. **Given** a pending invitation, **when** another account attempts to accept it, **then** the
   invitation remains pending and no pair is created.
3. **Given** a locked jar, **when** either partner submits a valid entry, **then** it is stored and
   the confirmation contains no entry text or readable entry metadata.
4. **Given** a locked jar with entries, **when** either partner requests entries, including entries
   they wrote, **then** no entry content, count, or identifying metadata is returned.
5. **Given** a locked jar, **when** either partner submits 100 valid entries on the same day within
   the configured short-window rate or retries throttled attempts after the indicated delay,
   **then** all 100 are accepted without a daily quota. The system need not accept 100 simultaneous
   first attempts without temporary throttling.
6. **Given** trusted system time is before a jar's effective unlock instant, **when** a partner
   requests its entries, **then** access is denied even one instant before unlocking.
7. **Given** trusted system time is at or after the effective unlock instant, **when** either
   partner requests the jar, **then** they can retrieve every accepted entry with its author and
   creation time without waiting for another action to open the jar.
8. **Given** an unlocked jar, **when** a partner tries to add an entry, **then** the write is
   rejected and the existing collection remains unchanged.
9. **Given** an unlocked jar, **when** either partner starts a new jar, **then** the old jar remains
   readable and the new jar's entries remain locked until its own unlock instant.
10. **Given** any jar, **when** a nonmember attempts to read or write, **then** access is denied
    without revealing entries or pair membership.
11. **Given** the current jar has unlocked, **when** both partners concurrently attempt to start
    the next jar, **then** exactly one new locked jar is created for the pair.
12. **Given** an unpaired account has multiple active invitations, **when** it accepts one and joins
    a pair, **then** its other incoming and outgoing invitations are invalidated and cannot be
    delivered, retried, accepted, or form another pair.
13. **Given** a verified unpaired account, **when** it creates an invitation, **then** the invitation
    and its delivery request are reliably recorded together and an accepted response reports
    pending-delivery status without revealing whether the target email belongs to an account.
14. **Given** an invitation awaiting delivery, **when** email delivery succeeds or fails, **then**
    its status becomes `PENDING` or `DELIVERY_FAILED`, respectively, and the inviter can observe
    the result.
15. **Given** a `DELIVERY_FAILED` invitation, **when** the inviter requests a retry, **then** the
    same invitation returns to `PENDING_DELIVERY` and no duplicate invitation is created.
16. **Given** an invitation in `PENDING_DELIVERY` or `DELIVERY_FAILED`, **when** anyone attempts to
    accept it, **then** no pair or initial jar is created.
17. **Given** an invitation has reached seven days from creation, **when** delivery processing,
    retry, or acceptance is attempted, **then** the attempt is rejected and the inviter must create
    a new invitation.
18. **Given** an inviter already has an unexpired invitation for the same normalized target email in
    `PENDING_DELIVERY`, `PENDING`, or `DELIVERY_FAILED`, **when** another creation request is made,
    **then** it is rejected without creating another invitation or sending another email.
19. **Given** an active invitation, **when** cancellation races with acceptance, **then** only one
    of the mutually exclusive terminal outcomes `ACCEPTED` or `CANCELLED` commits. If cancellation
    commits while delivery or retry work is queued or in progress, no later result can reactivate the
    invitation, and its link cannot create a pair even if email was already handed to the provider.
20. **Given** an email-verification token is expired or has been superseded, **when** it is submitted,
    **then** verification fails; when verification is resent, the same generic accepted response is
    returned regardless of account state, only the newest token can succeed, and invitation expiry
    remains unchanged.
21. **Given** an operation-specific short-window threshold is exceeded, **when** another covered
    request is made, **then** it receives a retryable response with an indicated delay without
    disclosing account, email, invitation, membership, or protected-resource existence. The attempt
    is not an accepted entry and may be retried after the delay; later valid entry writes remain
    unlimited by day.

---

### User Story 2 - Agree on a custom unlock time (Priority: P2)

Either partner can propose a different future unlock date. The change takes effect only after the
other partner approves it. The current date continues to govern access while approval is pending.

**Why this priority**: Couples can choose a meaningful reveal date without letting one person
unilaterally expose the other's entries.

**Independent Test**: With a locked jar, propose an earlier and a later future date in separate
tests; verify that neither proposal changes access until the other partner approves it.

**Acceptance Scenarios**:

1. **Given** a locked jar, **when** one partner proposes a future unlock date, **then** the
   existing unlock instant remains effective and the proposal is visible to both partners.
2. **Given** a pending proposal, **when** the other partner approves before the jar unlocks,
   **then** the proposed instant becomes the effective unlock instant for both partners.
3. **Given** a pending proposal, **when** the proposer tries to approve it or the other partner
   rejects it, **then** the current unlock instant remains unchanged.

### Edge Cases

- At the exact effective unlock instant, reads are allowed; before that instant they are denied.
  This behavior does not depend on a separate transition occurring at that instant.
- A proposal whose date is no longer in the future at approval is rejected. A proposal pending when
  the jar unlocks expires, and an unlocked jar cannot be relocked or rescheduled.
- Invitations that expire, are canceled, are already accepted, target the inviter, or target a
  paired account cannot create a pair. An unpaired account may have multiple active invitations;
  after it joins a pair, its other active incoming or outgoing invitations are invalidated.
  Concurrent acceptances involving the same account cannot create conflicting pairs. An account
  with an email address different from the invited address cannot accept.
- An inviter may have multiple active invitations only when they target different normalized email
  addresses. `PENDING_DELIVERY`, `PENDING`, and `DELIVERY_FAILED` are active states. A duplicate
  creation request for the same inviter and normalized target is rejected without creating another
  invitation or delivery message. A new invitation may be created after the previous one is
  accepted, canceled, expired, or invalidated if the inviter remains eligible.
- The inviter may cancel an invitation in any active state. `ACCEPTED` and `CANCELLED` are mutually
  exclusive terminal outcomes, so concurrent acceptance and cancellation allow only one to commit.
  Once cancellation commits, no new acceptance or delivery retry succeeds, queued delivery is
  skipped, and an in-flight delivery result or previously queued retry cannot return the invitation
  to an active state. Email handed to the mail provider immediately before cancellation cannot be
  recalled, but its invitation link remains unusable.
- An invitation with a missing or invalid time zone is rejected. Invitation email is delivered
  asynchronously: while delivery is pending or after it fails, the invitation cannot be accepted
  and no pair is created. A failed delivery is visible to the inviter and can be retried on the
  same invitation before expiry. Delivery and retry processing stop once seven days have elapsed
  from creation, and neither operation extends the deadline.
- A request for an unauthorized pair, jar, entry, or related protected resource cannot reveal its
  existence; access to an existing unauthorized resource is externally indistinguishable from
  access to a nonexistent one.
- A partner who signs out cannot keep using the ended session to view settings or entries.
- Multiple writes near the unlock boundary are accepted only while the jar is still locked; an
  accepted write must appear in the unlocked collection exactly once.
- Duplicate or concurrent attempts to start the next jar after unlocking cannot create more than
  one locked jar for the pair.
- Calendar year rollover and daylight-saving changes use the jar's recorded time zone to determine
  the unlock instant. The time zone stays fixed for that jar.
- Large jars can be read in parts after unlocking without missing or repeating entries, even if
  both partners read concurrently.
- Empty or oversized entries receive validation errors without storing partial content.
- Email-verification tokens expire 24 hours after issuance. A resend request always returns the same
  generic accepted response whether the email is absent, unverified, or already verified. Issuing a
  replacement supersedes every prior unsuperseded token, including expired tokens, concurrent resends
  leave only one usable newest token, raw tokens are never stored or logged, and resending does not
  change invitation expiry.
- Authentication attempts, verification resend, invitation creation and delivery retry, and
  entry-write requests use required, configurable, operation-specific short-window abuse throttles.
  Throttled responses identify when retry is permitted without disclosing whether an account, email,
  invitation, membership, or protected resource exists. Throttling never creates a daily entry quota
  or permanently reduces daily entry capacity.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST let people register with a unique email address, verify ownership
  of that address, sign in, and sign out. All pair, jar, setting, and entry operations MUST
  require authentication. An email-verification token MUST expire 24 hours after issuance. An
  unverified account MUST be able to request a replacement through a resend operation that returns
  the same generic accepted response whether the email is absent, unverified, or already verified.
  Issuing a replacement MUST supersede every prior unsuperseded verification token, including expired
  tokens, so only the newest unexpired token can be used. Concurrent resend requests MUST NOT leave
  multiple usable tokens. The verification secret MUST appear only in the intended verification
  message delivered to the account owner; registration and resend responses MUST NOT return it. Raw
  verification tokens MUST NOT be stored or exposed in logs, metrics, events, errors, or audit
  records. Replacement delivery MUST use the system's reliable email-delivery mechanism.
  Verification resend MUST NOT reset or otherwise modify invitation expiration.
- **FR-002**: An unpaired verified account holder MUST be able to invite a partner by email,
  whether or not the invitee has registered. The invitation MUST include a selected time zone.
  Creation MUST reliably and atomically record the invitation in `PENDING_DELIVERY` status and its
  delivery request. It MUST then return an accepted response with status `PENDING_DELIVERY` without
  waiting for email delivery. Successful delivery MUST change
  the status to `PENDING`; failed delivery MUST change it to `DELIVERY_FAILED`. The inviter MUST be
  able to observe these statuses and request delivery retry for the same invitation; retry MUST NOT
  create a duplicate invitation. An inviter MUST have at most one active, unexpired invitation for
  the same normalized target email; `PENDING_DELIVERY`, `PENDING`, and `DELIVERY_FAILED` are active
  states. A duplicate creation request MUST be rejected without creating another invitation or
  delivery message. Only a delivered invitation in `PENDING` status MAY be accepted, and only by an
  account that verifies the invited address. `PENDING_DELIVERY` and `DELIVERY_FAILED` invitations
  MUST NOT create a pair. An invitation MUST expire after seven days from its creation time unless
  accepted or canceled sooner by the inviter. Delivery and delivery retries MUST NOT reset or extend
  the expiration time. An expired invitation MUST NOT be delivered, retried, or accepted; the inviter MUST
  create a new invitation. After an invitation is accepted, canceled, expired, or invalidated, a
  new invitation to the same target MAY be created if the inviter remains eligible. Invitation
  requests and responses MUST NOT disclose whether the invited address already has an account.
  The inviter MAY cancel an invitation in `PENDING_DELIVERY`, `PENDING`, or `DELIVERY_FAILED`.
  `ACCEPTED` and `CANCELLED` MUST be mutually exclusive terminal outcomes; concurrent acceptance and
  cancellation MUST allow only one to commit. Once cancellation commits, no acceptance or delivery
  retry may succeed, queued delivery MUST be skipped, and an in-flight delivery result or retry
  queued before cancellation MUST NOT return the invitation to an active state. If email was handed
  off before cancellation completed, it cannot be recalled but its invitation link MUST remain
  unusable.
- **FR-003**: A pair MUST contain exactly two distinct verified accounts. An account MUST NOT
  belong to more than one active pair, and an established jar MUST NOT permit partner replacement.
  Subject to FR-002's per-target uniqueness rule, an unpaired account MAY have multiple active
  invitations. Once it joins a pair, all other active invitations involving that account MUST be
  invalidated and MUST NOT be delivered, retried, accepted, or create another pair. Duplicate or
  concurrent invitation acceptances MUST NOT create conflicting pairs.
- **FR-004**: Accepting an invitation MUST create a private locked jar for the pair. The invitee
  MUST see and accept the valid selected time zone before pairing. The pair MUST record that
  agreed time zone, and each of its jars MUST use it as a fixed time zone.
- **FR-005**: A new jar's default unlock instant MUST be January 1 at 00:00:00 of the calendar
  year immediately following the jar's creation year, interpreted in its fixed recorded time
  zone. This preserves the end-of-year reveal and MUST be strictly later than jar creation.
- **FR-006**: Either partner MUST be able to submit a nonempty text entry of at most 5,000
  characters to their locked jar. There MUST be no per-day entry count limit. Each accepted entry
  MUST retain its exact text, author, creation time, and jar association.
- **FR-007**: A successful entry submission MUST acknowledge storage without returning the stored
  text, another entry, or readable entry metadata. There MUST be no entry list, detail, search,
  export, preview, or equivalent read access to a locked jar, including for an entry's author.
- **FR-008**: A jar MUST be treated as LOCKED when trusted system time is earlier than its
  effective unlock instant and UNLOCKED when trusted system time is at or after that instant.
  Lock-dependent behavior MUST follow this comparison without requiring a separate transition.
  While LOCKED, every request to read its entries MUST be denied without returning entry content,
  entry counts, or identifying entry metadata. This rule MUST apply to direct identifiers,
  collection reads, error responses, and any alternate read surface; requester-supplied time
  MUST NOT influence the decision.
- **FR-009**: At and after the effective unlock instant, each partner MUST be able to retrieve the
  complete set of accepted entries with exact text, author, and creation time. Large collections
  MAY be delivered in parts, but every entry MUST be reachable in a stable order without loss or
  duplication.
- **FR-010**: Only the two members of a pair MUST be able to write to or read from its jars.
  Nonmembers and unauthenticated requesters MUST be denied before and after unlocking. Requests
  for existing protected pairs, jars, entries, or related resources that a requester is not
  authorized to access MUST be externally indistinguishable from requests for nonexistent ones;
  neither existence nor membership may be disclosed.
- **FR-011**: Both partners MUST be able to view the jar's effective unlock instant, time zone,
  lock status, and pending proposal without revealing entries. Either partner MUST be able to
  propose a new unlock instant that is strictly in the future. The proposal MUST identify the
  proposed instant and its proposer and MUST NOT alter the effective unlock instant until approved
  by the other partner.
- **FR-012**: Every unlock-date change, whether earlier or later, MUST have approval from both
  partners: one proposes and only the other MAY approve or reject. Approval MUST be rejected if
  the proposed instant is no longer in the future or the jar has unlocked. Only one proposal MAY
  be pending per jar; the proposer MAY cancel it, and a new proposal requires resolution or
  cancellation of the old one. Approved changes MUST take effect consistently for both partners.
- **FR-013**: At and after its unlock instant, a jar MUST remain readable to its pair and MUST
  reject further writes and date changes. It MUST NOT be reopened for writing. Either partner MAY
  start a separate locked jar for a new writing period, with access for both partners and the
  pair's agreed time zone. If both partners start the next jar concurrently, exactly one new jar
  MUST be created. A pair MUST have at most one LOCKED jar at a time; duplicate or concurrent
  requests MUST NOT create multiple locked jars.
- **FR-014**: The system MUST protect credentials and entry content in storage and transit and
  MUST NOT place entry text, credentials, or invitation secrets in routine logs, error messages,
  confirmations, or reminders. Any privileged operational access to stored entry text before
  unlock MUST be restricted to authorized personnel and audited. Rejected access and failures MUST
  be diagnosable without sensitive content.
- **FR-015**: Validation and authorization failures MUST produce consistent, actionable errors
  without revealing protected data. Concurrent writes, pairing, date approvals, and unlock checks
  MUST preserve the membership, entry, and effective-date rules above.
- **FR-016**: The system MUST apply configurable, operation-specific short-window abuse throttles to
  authentication attempts, email-verification resend, invitation creation and delivery retry, and
  entry-write requests. A throttled request MUST receive a retryable response indicating the delay
  before another attempt. Throttling MUST NOT reveal whether an account, email address, invitation,
  membership, or protected resource exists. It MUST NOT impose a daily entry quota or permanently
  reduce the number of entries a couple may submit. A throttled entry attempt is not accepted and
  MUST be permitted to retry after the indicated delay.

### Key Entities *(include if feature involves data)*

- **Account**: A person with a unique verified email address and authentication state; may belong
  to one active pair.
- **Invitation**: A time-limited request from one account to a designated second account, with
  a target email address and `PENDING_DELIVERY`, `PENDING`, `DELIVERY_FAILED`, accepted, canceled,
  expired, or invalidated status. Only `PENDING` means delivery succeeded and permits acceptance.
  An account may have multiple active invitations while unpaired, but an inviter has at most one
  unexpired active invitation per normalized target email. Joining a pair invalidates its other
  active invitations. `ACCEPTED` and `CANCELLED` are mutually exclusive terminal outcomes, and no
  delivery result or retry may return a canceled or expired invitation to an active state.
- **Pair**: Exactly two accounts that share access to their jars and an agreed time zone.
- **Jar**: A pair-owned collection with a fixed recorded time zone and effective unlock instant.
  LOCKED or UNLOCKED behavior derives from comparing trusted system time with that instant; a
  separate status is not the source of truth. A pair may have past unlocked jars and at most one
  currently locked jar.
- **Entry**: A text note associated with one jar and author, plus its creation time. It is
  unreadable through the system while its jar is locked.
- **Unlock Proposal**: A proposed future unlock instant, proposer, other partner's decision, and
  resolution status. Only an approved proposal changes the jar's effective unlock instant.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In acceptance and security tests, 100% of user-facing attempts to read a locked jar
  return no entry content, count, or identifying entry metadata, including at the instant
  immediately before unlock.
- **SC-002**: In acceptance and security tests, 100% of unauthorized or unauthenticated requests
  for protected pairs, jars, entries, or related resources are denied without disclosing existence
  or membership; an existing unauthorized resource and a nonexistent one are externally
  indistinguishable to the same requester.
- **SC-003**: Every accepted entry in a jar containing at least 10,000 entries is retrievable
  exactly once after unlock, with unchanged text and correct author and creation time; 100 valid
  writes on one day are accepted without a daily cap when submitted within the configured
  short-window rate or retried after any indicated throttling delay. This does not require 100
  simultaneous first attempts to succeed.
- **SC-004**: In all date-change tests, a unilateral or pending proposal never changes access;
  an approved future proposal changes the effective unlock instant for both partners.
- **SC-005**: In a documented, representative test environment and workload that specifies jar
  sizes, request mix, and 100 concurrently active pairs, at least 95% of valid writes receive a
  storage confirmation within two seconds, and at least 95% of post-unlock reads show their first
  entries within three seconds. These are targets for that test scenario, not universal latency
  guarantees.
- **SC-006**: After a web or mobile client is available, an external moderated pairing trial should
  demonstrate that at least 90% of invited partners can register, verify their invited email, and
  accept within ten minutes of invitation delivery without assistance. This deferred product-level
  usability validation is outside the backend acceptance gate and does not block backend completion.
- **SC-007**: In all duplicate and concurrent invitation-acceptance and next-jar creation tests,
  no account joins more than one pair, and simultaneous next-jar attempts create exactly one jar
  with no second locked jar for the pair.
- **SC-008**: In all invitation-delivery lifecycle tests, creation reliably records exactly one
  invitation and delivery request and returns an accepted response with `PENDING_DELIVERY`; delivery
  of an eligible active invitation produces `PENDING` or `DELIVERY_FAILED`; retry reuses the
  invitation; terminal invitations never return to active; and no status other than `PENDING` can
  create a pair or initial jar.
- **SC-009**: In abuse-throttling tests, every throttled covered operation returns a retryable delay
  without account, email, invitation, membership, or protected-resource existence disclosure, and
  valid entry submissions resume after that delay without a daily quota or permanent reduction in
  the number of entries the couple may submit.

## Assumptions

- This specification covers the backend and its externally visible behavior. Web and mobile user
  interfaces, push notifications, and writing reminders are outside the initial release.
- SC-006 is retained for product traceability but is evaluated only after a web or mobile client is
  available; it is not part of the backend Definition of Done.
- Writing is available at any time while a jar is locked, with no daily quota.
- Entry text is plain text. Attachments, edits, deletion, anonymous entries, account deletion,
  pair dissolution, and partner replacement are outside this feature.
- A partner necessarily knows text they are currently composing. The backend must prevent later
  retrieval of stored entries, including the author's own submissions, until unlocking.
