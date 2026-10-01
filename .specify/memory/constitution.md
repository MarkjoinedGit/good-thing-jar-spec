<!--
Sync Impact Report
- Version change: 1.0.0 -> 1.1.0 (MINOR: extend scope and add frontend obligations).
- Amendment reason: govern both Good Thing Jar applications while preserving business rules,
  security requirements, backend obligations, and backward-compatible API contracts.
- Modified principles:
  - I. Correctness First -> I. Correctness First (Shared): clarify preservation of domain rules.
  - II. Clean Architecture -> II. Clean Architecture (Backend Only): retain backend boundaries.
  - III. API Contracts -> III. API Contracts (Shared; Backend Duties Identified).
  - IV. Persistence & Transactions -> IV. Persistence & Transactions (Backend Only).
  - V. Testing -> V. Testing (Shared): add frontend critical-behavior coverage.
  - VI. Security & Observability -> VI. Security & Observability (Shared): add browser privacy
    and session cleanup requirements.
  - VII. Maintainability -> VII. Maintainability (Shared): unchanged obligations.
  - VIII. Definition of Done -> VIII. Definition of Done (Shared; Application-Specific Gates).
- Added principles: IX. Frontend Architecture; X. Backend Authority;
  XI. Responsive & Accessible Interaction.
- Added sections: Scope & Repository Boundaries.
- Modified sections: Development Workflow; Governance (scope and amendment compliance).
- Removed sections: none.
- Templates requiring updates:
  - ✅ updated: .specify/templates/plan-template.md
  - ✅ updated: .specify/templates/spec-template.md
  - ✅ updated: .specify/templates/tasks-template.md
  - ✅ updated: .specify/templates/agent-file-template.md
  - ✅ updated: .agents/skills/speckit-tasks/SKILL.md (local command guidance).
  - ✅ updated: .agents/skills/speckit-implement/SKILL.md (completion guidance).
- Runtime guidance: ✅ updated: AGENTS.md; backend feature quickstart reviewed, still applicable.
- Other templates: constitution-template.md and checklist-template.md reviewed; no changes needed.
- Command templates: .specify/templates/commands/ absent; local plan, task, and implementation
  skill guidance reviewed instead.
- Follow-up TODOs: none.
-->
# Good Thing Jar Constitution

## Scope & Repository Boundaries

This constitution governs both the Good Thing Jar backend and frontend. Principles labeled Shared
apply to both applications. Backend Only principles govern backend implementation; Frontend Only
principles govern frontend implementation. Application-specific duties inside shared principles are
explicitly labeled and MUST be assessed for the affected application.

Specifications, plans, tasks, contracts, and other planning documents MUST remain in
`C:\workspace\good-thing-jar\good-thing-jar-spec`. Backend implementation MUST remain in the sibling
`C:\workspace\good-thing-jar\good-thing-jar-backend` directory. Frontend implementation MUST remain in
the separate sibling `C:\workspace\good-thing-jar\good-thing-jar-front-end` directory. Build, run,
test, migration, and source-generation commands MUST execute from the application directory they
affect; generated implementation artifacts MUST stay there.

## Core Principles

### I. Correctness First (Shared)

Changes MUST preserve business correctness and data integrity. Implement the smallest change that
satisfies the specification. Existing behavior MUST remain unchanged unless the specification
explicitly requires a change. This limits regressions and protects established business rules.

Both applications MUST preserve the specified pairing, invitation, entry privacy, unlock-time,
mutual approval, read-only opened jar, and next-jar rules. Frontend flows MUST NOT introduce a
different interpretation of those rules or weaken their enforcement.

### II. Clean Architecture (Backend Only)

API, business, and persistence concerns MUST remain separate. Controllers MUST delegate business
decisions to the business layer, and persistence concerns MUST stay out of the API layer. Designs
SHOULD favor simple, loosely coupled components and composition; inheritance or abstraction MUST
have a concrete need. These boundaries keep behavior understandable and independently testable.

### III. API Contracts (Shared; Backend Duties Identified)

Backend: Public APIs MUST use appropriate request and response DTOs and MUST NOT expose persistence
entities.
External input MUST be validated, and errors MUST follow a consistent, centralized handling policy.
Existing API contracts MUST remain backward compatible unless a breaking change is explicitly
requested. Stable contracts protect clients from internal persistence changes.

Frontend: API access MUST follow the documented request, response, validation, and error contracts.
Client-side validation MUST NOT replace backend validation. Frontend changes MUST NOT silently
require incompatible API behavior; any contract change MUST follow the compatibility rule above.

### IV. Persistence & Transactions (Backend Only)

Database access, JPA relationships, fetch strategies, and transaction boundaries MUST be deliberate.
Changes MUST avoid N+1 queries, unnecessary database access, and correctness that depends on an
accidental persistence-context state. Transactions and concurrent operations MUST protect data
integrity. Database schema changes MUST use version-controlled migrations. These rules make data
behavior predictable under load and concurrent use.

### V. Testing (Shared)

Business-critical behavior and bug fixes MUST have appropriate automated tests. Use unit tests for
isolated logic and integration tests when Spring, JPA, database, transaction, or API behavior matters.
Tests MUST verify observable behavior rather than implementation details. This gives evidence that
the required behavior works and guards against regression. Spring, JPA, database, and transaction
test duties apply to the backend.

Frontend automated tests MUST cover business-critical behavior, particularly authentication,
privacy, and unlock boundaries. Coverage MUST include session termination and private-state cleanup,
denied access, browser clock changes and countdown expiry without backend permission, and behavior
immediately before and at unlock. Use unit, component, integration, or browser tests according to
the behavior involved, and verify relevant user journeys against the real backend.

### VI. Security & Observability (Shared)

Secrets and sensitive information MUST NOT be hardcoded or exposed. Authentication, authorization,
input validation, and logging MUST be applied where the behavior requires them. Failures MUST be
diagnosable without leaking sensitive data. These safeguards protect users and support production
incident investigation.

Frontend: Tokens, entry content, and other sensitive data MUST NOT be exposed through logs,
analytics, or persistent caches. Browser storage and any persisted query or service-worker cache
MUST NOT retain those values. Logout or session termination, including expiry or revocation detected
by the client, MUST clear private application state and query caches. Pending requests MUST NOT
repopulate private state after termination or expose a previous user's data to a subsequent session.
These rules protect privacy on shared browsers and after access ends.

### VII. Maintainability (Shared)

Code MUST favor readability and simplicity over cleverness or premature abstraction. Follow existing
project conventions before adding patterns or dependencies. Changes MUST avoid unrelated refactoring.
Comments and documentation SHOULD explain non-obvious decisions and reasoning rather than restate
code. Focused, conventional changes are easier to review and maintain.

### VIII. Definition of Done (Shared; Application-Specific Gates)

A change is complete only when the specification is satisfied, relevant tests pass, data integrity
and security are preserved, and no unintended behavior is introduced. Backend completion also
requires consideration of persistence and transaction implications. Reviewers MUST verify these
conditions before accepting a change.

Frontend completion additionally requires passing type checking, linting, a production build, and
relevant automated tests, plus verification of affected flows against the real backend. The initial
prototype MUST be fully usable and testable in a desktop browser. Responsive mobile behavior,
keyboard navigation, form labels, and interaction states MUST be verified for affected flows.
Plans and review evidence MUST record the commands, backend environment, scenarios, and results;
mock-only verification does not satisfy the real-backend gate.

### IX. Frontend Architecture (Frontend Only)

Presentation, application flows, and API access MUST remain separate. UI components MUST delegate
flow coordination to application logic and network calls to an API access boundary. Designs SHOULD
use a simple feature-based structure; another structure requires a concrete justification in the
plan. Avoid layers or abstractions without a current need. These boundaries keep user journeys and
API behavior understandable and independently testable.

### X. Backend Authority (Shared; Backend and Frontend Duties Identified)

Backend: The backend MUST remain authoritative for authentication, authorization, and jar lock status.
Jar access MUST follow the specified effective unlock instant and trusted backend time, including
the exact unlock boundary. Backend enforcement MUST hold regardless of client state or requests.

Frontend: Browser clocks and countdowns MUST be display-only. Local time, UI state, route guards,
and countdown expiry MUST NOT grant access, reveal stored entries, or permit operations the backend
denies. At countdown expiry or after stale state, the frontend MUST obtain a backend decision before
showing an unlocked jar or protected content, and MUST handle denials without sensitive-data exposure.
This preserves privacy when a browser clock is wrong or a session or unlock proposal changes.

### XI. Responsive & Accessible Interaction (Frontend Only)

The frontend MUST support desktop and mobile browsers through responsive layouts. The initial
prototype MUST allow all included user journeys to be completed and tested in a desktop browser.
Affected flows MUST support keyboard navigation, visible keyboard focus, and accessible form labels.
Loading, empty, error, and submission states MUST be meaningful and communicate progress, outcomes,
and available recovery actions without leaking sensitive data. These requirements make the product
usable across screen sizes and input methods.

## Decision Priority

When principles conflict, resolve the tradeoff in this order:
Correctness & Data Integrity > Security > Maintainability > Testability > Performance > Convenience.
A lower-priority benefit MUST NOT override a higher-priority obligation without an explicit
amendment to this constitution.

## Development Workflow

Specifications MUST identify intended behavior and any API, validation, compatibility, data, or
security effects, and identify the applications affected. Plans MUST assess applicable architecture,
API compatibility, test scope, and observability. Backend plans MUST assess DTO boundaries,
persistence access, transactions, and migrations. Frontend plans MUST assess feature structure,
backend authority, session and cache cleanup, responsive accessibility, interaction states, and
real-backend verification. Tasks MUST include the implementation and verification work needed to
meet the Definition of Done. Reviews MUST check the stated contract
against the actual change and test results; unresolved risks MUST be documented before acceptance.

## Governance

This constitution governs project specifications, plans, tasks, implementation, and review. Amendments
MUST be documented in this file with a reason, impact on dependent templates and runtime guidance,
and a version change. Amendments MUST preserve existing business rules, security requirements, and
API compatibility obligations unless their change is explicitly requested and reviewed. Each
amendment MUST update affected templates and guidance and include a Sync Impact Report; deferred
updates MUST be explicitly identified with a reason.
The version follows semantic versioning: MAJOR for incompatible principle or governance changes,
MINOR for new principles or materially expanded guidance, and PATCH for non-semantic clarification.
The ratification date remains the initial adoption date; the last-amended date changes on every
amendment. Every change review MUST check compliance with this constitution, and exceptions MUST be
explicitly justified and approved as part of the change review.

**Version**: 1.1.0 | **Ratified**: 2026-09-18 | **Last Amended**: 2026-10-01
