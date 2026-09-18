# Phase 1 Data Model: Shared Locked Jar

## Conventions

- Use application-generated UUID identifiers so modules can exchange IDs without loading entities.
- Store absolute timestamps as UTC-capable instants and store IANA zone IDs as text.
- Store normalized email separately from any presentation form; compare uniqueness using the
  normalized value.
- Store passwords and bearer/verification/invitation tokens only as one-way hashes.
- Include `created_at` and `updated_at` where shown; generate them from trusted server time.
- Use optimistic `version` fields on mutable aggregates. Use explicit pessimistic locks only for the
  low-frequency transitions described below.
- JPA entities and repositories remain internal to their owning module. Cross-module references are
  scalar IDs, not entity associations.

## Identity module

### Account

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `email_normalized` | text | Required; unique; canonical comparison value |
| `email_display` | text | Required; validated email returned only to its owner |
| `password_hash` | text | Required; adaptive one-way hash; never returned or logged |
| `email_verified_at` | instant, nullable | Set once after successful verification |
| `created_at` | instant | Required |
| `updated_at` | instant | Required |
| `version` | integer | Optimistic concurrency |

**Invariants**:

- One account per normalized email.
- Only verified accounts can send or accept pair invitations.

### EmailVerification

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `account_id` | UUID | Required; references Account |
| `token_hash` | text | Required; unique; never expose after creation |
| `expires_at` | instant | Required |
| `consumed_at` | instant, nullable | Single-use marker |
| `superseded_at` | instant, nullable | Set when a newer token is issued |
| `created_at` | instant | Required |

**State transitions and invariants**:

- A token is usable only when it is the newest unsuperseded token for the account, has not been
  consumed, and trusted time is earlier than `expires_at`.
- `expires_at` is exactly 24 hours after issuance. Active → consumed, active → superseded, and active
  → expired by comparison with trusted time are one-way transitions.
- Resend locks the account, supersedes every prior unsuperseded token, including expired tokens, and
  inserts the replacement and its outbox message atomically. Concurrent resends leave only one usable
  newest token.
- A partial unique index on `account_id` where `consumed_at IS NULL AND superseded_at IS NULL` is the
  final race guard. Expired records are superseded before a replacement insert.
- Only token hashes are persisted in verification records. The verification secret appears in clear
  form only in the intended account-owner message after mail-path decryption; it is never stored or
  exposed through API responses or diagnostic surfaces.
- Resend does not read or update invitation expiry and returns one generic response for absent,
  unverified, and already verified email addresses.

### AuthSession

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key; session-family identifier |
| `account_id` | UUID | Required; references Account |
| `access_token_hash` | text | Required; unique |
| `refresh_token_hash` | text | Required; unique |
| `access_expires_at` | instant | Required |
| `refresh_expires_at` | instant | Required |
| `revoked_at` | instant, nullable | Set on logout or security revocation |
| `created_at` | instant | Required |
| `last_used_at` | instant | Required |
| `version` | integer | Protects refresh rotation/reuse detection |

**Invariants**:

- Only an unrevoked session before its relevant expiry authenticates a request.
- Refresh rotates both token secrets atomically; reuse revokes the session family.
- Logout sets `revoked_at`, making the ended session unusable immediately.

## Pairing module

### Invitation

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `inviter_account_id` | UUID | Required; verified, unpaired account at creation |
| `target_email_normalized` | text | Required; cannot equal inviter email |
| `time_zone_id` | text | Required, valid IANA ID |
| `status` | enum | `PENDING_DELIVERY`, `PENDING`, `DELIVERY_FAILED`, `ACCEPTED`, `CANCELLED`, `EXPIRED`, `INVALIDATED` |
| `delivery_attempts` | integer | Nonnegative |
| `expires_at` | instant | Seven days after creation |
| `accepted_by_account_id` | UUID, nullable | Must own target email when accepted |
| `accepted_at` | instant, nullable | Set with `ACCEPTED` |
| `created_at` | instant | Required |
| `updated_at` | instant | Required |
| `version` | integer | Optimistic concurrency |

**Invariants**:

- Multiple pending invitations may involve an unpaired account.
- Creation inserts the invitation as `PENDING_DELIVERY` and its outbox message in one transaction.
- `expires_at` is fixed at exactly seven days after `created_at`; delivery and retry never change it.
- `PENDING_DELIVERY`, `PENDING`, and `DELIVERY_FAILED` are the only active statuses.
- An inviter has at most one active invitation per normalized target email. Before insert, the
  transaction locks and changes any matching active row whose deadline has passed to `EXPIRED`.
- A partial unique index on `(inviter_account_id, target_email_normalized)` for the three stable
  active statuses is the final race guard; it never uses current time in its predicate.
- Successful delivery changes `PENDING_DELIVERY` to `PENDING`; failed delivery changes it to
  `DELIVERY_FAILED`.
- Only a delivered `PENDING` invitation may be accepted. `PENDING_DELIVERY` and `DELIVERY_FAILED`
  cannot create a Pair or initial Jar.
- An inviter-requested retry changes the same `DELIVERY_FAILED` invitation back to
  `PENDING_DELIVERY` and queues delivery without inserting another invitation.
- The inviter may cancel from every active status. `ACCEPTED` and `CANCELLED` are mutually exclusive
  terminal outcomes; acceptance and cancellation lock the same invitation so only one commits.
- Once cancellation commits, queued delivery is skipped and acceptance or retry fails. A delivery
  result or retry queued before cancellation may record its outbox outcome but cannot change the
  invitation from `CANCELLED` back to an active status.
- Delivery, retry, and acceptance reject an invitation at or after `expires_at`. Late delivery
  results cannot change `EXPIRED` back to an active status.
- Acceptance requires a verified account whose normalized email matches the target.
- Joining a pair invalidates every other pending incoming or outgoing invitation involving either
  new member.
- `EXPIRED` is derived for behavior when trusted time reaches `expires_at`; it may be recorded during
  access or cleanup, but a delayed cleanup never makes an expired invitation acceptable.
- After `ACCEPTED`, `CANCELLED`, `EXPIRED`, or `INVALIDATED`, a new invitation may be created for the
  same inviter and normalized target if the inviter remains eligible.

### Pair

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `time_zone_id` | text | Required, agreed IANA zone; immutable |
| `created_at` | instant | Required |
| `version` | integer | Locked when creating a next jar |

### PairMember

| Field | Type | Rules |
|---|---|---|
| `pair_id` | UUID | References Pair; part of primary key |
| `account_id` | UUID | References Account; part of primary key; globally unique |
| `joined_at` | instant | Required |

**Constraints and invariants**:

- Unique `account_id` enforces membership in at most one pair.
- A successful acceptance inserts exactly two members in the same transaction as Pair and initial
  Jar creation. The application rejects self-pairing and commits no partial pair.
- Pair membership is immutable in this feature.

## Jar module

### Jar

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `pair_id` | UUID | Required scalar reference to Pair |
| `sequence_number` | integer | Required; positive; unique within pair |
| `current` | boolean | Required; exactly one current writing-cycle record per pair |
| `time_zone_id` | text | Required immutable snapshot of pair zone |
| `effective_unlock_at` | instant | Required; strictly after `created_at` at creation/change approval |
| `created_by_account_id` | UUID | Required pair member |
| `created_at` | instant | Required |
| `version` | integer | Optimistic concurrency; exclusive lock for unlock-date approval |

**Derived behavior**:

- LOCKED iff `trustedClock.instant() < effective_unlock_at`.
- UNLOCKED iff `trustedClock.instant() >= effective_unlock_at`.
- `current` identifies the pair's latest writing cycle and is not lock status. The current jar can be
  UNLOCKED between its reveal and creation of the next jar.
- The default `effective_unlock_at` is January 1 00:00:00 of the year after the creation year in
  `time_zone_id`, converted to an absolute instant.

**Constraints and indexes**:

- Unique `(pair_id, sequence_number)`.
- Partial unique index on `pair_id WHERE current = true`.
- Index `(pair_id, created_at DESC)` for jar history.

### Entry

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key; never returned by the write response |
| `jar_id` | UUID | Required; references Jar |
| `author_account_id` | UUID | Required; must be a member of the jar's pair |
| `body` | text | Required; 1–5,000 Unicode characters after request validation |
| `created_at` | instant | Required trusted server time |

**Constraints and indexes**:

- Index `(jar_id, created_at, id)` supports stable keyset pagination.
- No update/delete behavior is exposed.
- Body, ID, counts, author, and timestamps are unavailable through user-facing reads while LOCKED.

### UnlockProposal

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `jar_id` | UUID | Required; references Jar |
| `proposed_by_account_id` | UUID | Required pair member |
| `proposed_unlock_at` | instant | Required; future at proposal and approval |
| `status` | enum | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`, `EXPIRED` |
| `resolved_by_account_id` | UUID, nullable | Other partner for approve/reject |
| `resolved_at` | instant, nullable | Required for resolved states |
| `created_at` | instant | Required |
| `version` | integer | Prevents duplicate resolution |

**Constraints and invariants**:

- Partial unique index on `jar_id WHERE status = 'PENDING'`.
- Proposer may cancel but may not approve or reject.
- Only the other partner may approve or reject.
- Approval atomically updates the jar's `effective_unlock_at` and proposal status.
- Pending proposals behave as expired when the jar becomes UNLOCKED, regardless of cleanup timing.

## Notification/platform records

### OutboxMessage

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `event_type` | text | Verification or invitation delivery only |
| `aggregate_id` | UUID | Account or invitation ID |
| `payload` | JSON | Minimal delivery data; never jar entry text; verification secret encrypted and access-controlled, never stored raw |
| `status` | enum | `PENDING`, `SENT`, `RETRYABLE_FAILURE`, `PERMANENT_FAILURE` |
| `attempt_count` | integer | Nonnegative |
| `next_attempt_at` | instant | Required for retryable work |
| `created_at` | instant | Required |
| `processed_at` | instant, nullable | Set on terminal success/failure |

For an eligible active invitation, processing `SENT` changes it to `PENDING`, while a failed delivery
outcome changes it to `DELIVERY_FAILED`. A user-requested retry may append a new outbox message for
the same invitation aggregate, but never creates a new Invitation. Once the invitation is
`DELIVERY_FAILED`, its failed outbox attempt is not automatically rescheduled; only the delivery-retry
operation queues another attempt. Invitation
workers skip queued work after cancellation or expiry. If external handoff completed immediately
before cancellation, the message cannot be recalled, but the conditional result update cannot change
the terminal invitation. Verification delivery clears its encrypted one-time secret after terminal
processing. Only the mail-delivery path may decrypt the secret to produce the intended account-owner
verification message; plaintext is never persisted or exposed through diagnostic surfaces.

### SecurityAuditEvent

| Field | Type | Rules |
|---|---|---|
| `id` | UUID | Primary key |
| `actor_type` | enum | `ACCOUNT`, `OPERATOR`, `SYSTEM` |
| `actor_reference` | text | Pseudonymous/account ID; no credentials or raw token |
| `action` | text | Authentication, denial, operational access, or security mutation |
| `resource_type` | text | Coarse category only |
| `resource_id` | UUID, nullable | Omit when unnecessary; never include entry body |
| `outcome` | enum | `SUCCESS`, `DENIED`, `FAILURE` |
| `correlation_id` | text | Request trace reference |
| `occurred_at` | instant | Required |

### AbuseThrottleBucket

| Field | Type | Rules |
|---|---|---|
| `operation` | enum/text | Authentication, verification resend, invitation creation, invitation delivery retry, or entry write |
| `scope_hash` | bytes/text | Required keyed one-way pseudonymous scope; never stores a raw email or token |
| `window_started_at` | instant | Required; part of primary key |
| `request_count` | integer | Required; atomically incremented and nonnegative |
| `expires_at` | instant | Required cleanup boundary after the window |

**Invariants**:

- Primary key `(operation, scope_hash, window_started_at)` makes the counter shared by every
  application instance.
- Capacity and window duration are configured independently for each operation.
- An atomic increment either admits the attempt or returns the remaining window as `Retry-After`.
- Throttle evaluation and response shape do not reveal account, email, invitation, membership, or
  protected-resource existence.
- Buckets count short-window attempts only; they are not daily entry counters and cannot permanently
  reduce the number of entries a couple may submit.

## Transaction and lock matrix

| Use case | Transaction boundary and concurrency control |
|---|---|
| Register / verify account | One identity transaction; unique normalized email; lock the account during verification so consume/resend races preserve newest-unexpired single-use token rules |
| Resend verification | Apply shared throttle; lock account when eligible; supersede prior tokens; insert one hashed replacement and outbox message atomically; never update invitations; return the generic accepted response for every account state |
| Apply abuse throttle | Atomically increment the PostgreSQL bucket for the operation and pseudonymous scope; on rejection return the remaining window without performing the protected operation |
| Create invitation | Apply shared throttle; lock inviter and matching active invitation; mark a stale match `EXPIRED`; persist one `PENDING_DELIVERY` invitation and outbox message atomically; normalize the active-status uniqueness race; return without waiting for SMTP |
| Record invitation delivery | Lock invitation; record the outbox outcome; conditionally set `PENDING` or `DELIVERY_FAILED` only when still `PENDING_DELIVERY` and before expiry; never reactivate a terminal invitation |
| Cancel invitation | Lock invitation; require inviter and any active status; commit `CANCELLED`; later delivery/retry results cannot reactivate it |
| Retry invitation delivery | Apply shared throttle; lock the same invitation; require its inviter, `DELIVERY_FAILED`, and time before expiry; set `PENDING_DELIVERY` and enqueue delivery atomically without inserting another invitation |
| Accept invitation | Lock invitation and both account rows in UUID order; require `PENDING` and time before expiry, then verify membership; race with cancellation so only `ACCEPTED` or `CANCELLED` commits; create Pair, two PairMembers, and initial Jar; invalidate other invitations; commit atomically |
| Submit entry | Apply shared throttle; verify member/current jar; shared pessimistic lock Jar; sample Clock once after lock; insert only if LOCKED |
| Read entries | Read-only transaction; requester-scoped membership check first; compare trusted time; keyset page query only if UNLOCKED |
| Propose date | Exclusive Jar lock; confirm LOCKED and no pending proposal; insert proposal |
| Resolve date proposal | Lock proposal then Jar; verify other partner and future instant; atomically update proposal and Jar |
| Create next jar | Lock Pair row; load current Jar; require UNLOCKED; mark old `current=false`, insert next `current=true`; unique partial index is final guard |
| Refresh/logout | Lock AuthSession; rotate hashes or set revocation atomically |

## Fetch and query policy

- Do not model `Jar.entries`, `Pair.jars`, or invitation collections as eagerly traversable entity
  collections.
- Use projections for jar settings, invitation summaries, and entry pages.
- Membership checks query only `PairMember` by account and pair IDs.
- Entry pages use `(created_at, id) > (:cursorCreatedAt, :cursorId)` with a bounded limit.
- Every protected by-ID query either includes the authorized pair/account scope or normalizes a later
  denial to the same external result as absence.
