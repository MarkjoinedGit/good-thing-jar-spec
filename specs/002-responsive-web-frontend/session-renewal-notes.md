# Session Renewal Review Notes

**Branch**: `002-responsive-web-frontend` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)
**Status**: Historical pre-planning review notes. The planning workflow ran on 2026-10-01;
selected decisions now live in [plan.md](plan.md) and the
[frontend interface contract](contracts/frontend-interface.md). Keep these notes as evidence of
the review, not a substitute for the full plan or a separate implementation instruction.

## Verified Backend Constraints

- [SessionService.refresh](../../../good-thing-jar-backend/src/main/java/com/goodthingjar/identity/application/SessionService.java)
  checks throttling before renewal, locks the matching current/previous refresh token's session,
  rotates both tokens on success, and commits session revocation when the previous token is reused.
- [AuthSessionEntity](../../../good-thing-jar-backend/src/main/java/com/goodthingjar/identity/persistence/AuthSessionEntity.java)
  checks access expiry separately from refresh expiry/revocation and replaces the access token
  during rotation. Old access requests can therefore fail after a successful renewal.
- [AuthSessionRepository](../../../good-thing-jar-backend/src/main/java/com/goodthingjar/identity/persistence/AuthSessionRepository.java)
  uses a pessimistic write lock for refresh lookup; two simultaneous uses do not provide two safe renewals.
- The [OpenAPI contract](../001-shared-locked-jar/contracts/openapi.yaml) documents refresh 200, 401,
  and 429, using the existing generic authentication problem and Retry-After. Transport failure is
  client uncertainty, not a new backend error classification.
- [AuthenticationIntegrationTest](../../../good-thing-jar-backend/src/test/java/com/goodthingjar/identity/AuthenticationIntegrationTest.java)
  already checks rotation, previous-token reuse rejection, rotated-access denial after reuse,
  and logout revocation. This amendment preserves those behaviors without changing backend code.

## Candidate Coordination Approach for Planning

During planning, assess this approach within the frontend application/session and API-access
boundaries in sibling `good-thing-jar-front-end`; presentation would consume safe recovery state.
These notes select no library or new backend capability. The specification governs required
behavior; concrete coordination mechanisms are decisions for `/speckit.plan`.

1. Give each local sign-in a session generation and each successfully installed token pair a
   credential revision. Tag authenticated requests with both so delayed results can be recognized.
   Keep tokens and coordination state in memory, never logs, analytics, URLs, or persistent caches.
2. Route all renewal triggers through one session-scoped coordinator. Store the in-flight renewal
   promise before sending the refresh request; other callers await that promise rather than send
   the same refresh token independently. Pause new protected work using the rejected credential
   revision while renewal is unresolved. Scope coordination to all callers sharing these tokens.
   This release does not share tokens across tabs; any later sharing requires shared coordination.
3. On a protected 401 from the current revision, initiate or join renewal if credentials exist.
   A 401 from an older revision must not start another refresh: a safe read may be reissued once
   with the installed revision, while a mutation is surfaced for explicit user action, never replayed.
   Do not loop indefinitely if a recovery read is rejected again.
4. On refresh 200, install the complete returned token pair/expiry values and advance the revision
   atomically, only if the originating local session is still active. All waiting callers observe
   that same result. Discard late results after logout, sign-in replacement, or termination.
5. On refresh 401, terminate the local session, clear private state/query caches/drafts and tokens,
   and show generic sign-in guidance. Do not distinguish expiry, revocation, invalidity, or reuse.
   If renewal credentials are absent, require sign-in rather than infer a new backend error reason.
6. On explicit refresh 429, record one shared operation cooldown from Retry-After. Do not start
   parallel refreshes or treat this as revocation; retain the draft in the current session. After
   the delay, a user-requested attempt still goes through the same coordinator using the latest
   available credentials. A subsequent result is not guaranteed, and no mutation is replayed.
7. On network/timeout uncertainty, retain an unresolved recovery state rather than returning to
   normal protected access or reopening same-token retries. Rotation may already have committed.
   Offer explicit sign-in recovery; selecting it ends local state and clears drafts. Do not claim
   backend revocation or promise recovery of the old session. No undocumented refresh grace period,
   idempotency key, status endpoint, or token introspection is introduced.
8. Coordinate outcome state as well as the in-flight request. Clearing a rejected/throttled/uncertain
   promise must not let another waiting caller immediately start a refresh with the same token.
   Logout invalidates the coordinator and all queued continuations; late completion cannot restore
   access. Keep existing local cleanup even if remote logout cannot be confirmed.

Safe read recovery is a frontend policy, not a backend retry guarantee. Renewing credentials never
automatically replays POST/DELETE operations, including entry creation, proposals, invitations,
next-jar creation, or logout. Explicit user actions retain their existing backend semantics.

## Verification to Include in the Full Plan

- Protected 401 followed by refresh 200 retains the session/draft and adopts both rotated tokens.
- Several simultaneous protected 401 responses and another renewal trigger make one refresh request,
  share one result, and do not reuse the previous token. A delayed old-access 401 makes no extra renewal.
- Refresh 401 performs generic termination; refresh 429 observes the shared Retry-After cooldown;
  a lost success response causes no blind old-token retry and offers sign-in recovery.
- Logout/sign-in replacement during renewal prevents late credentials or query results restoring data.
- Recovery never automatically replays any mutation, even when a protected 401 was received.
- Verify actual backend rotation/reuse/revocation behavior and browser recovery on both desktop and
  mobile. The reuse test is deliberate test setup, not normal frontend behavior.

These are required future checks, not tests claimed to have run in this documentation amendment.
The full plan must additionally define frontend type checking, linting, production build, relevant
automated tests, and real-backend verification commands from the frontend directory, as required
by constitution v1.1.0. Backend source, contract, token lifetimes, locks, reuse detection, and
revocation behavior are unchanged.
