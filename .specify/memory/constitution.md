<!--
Sync Impact Report
- Version change: template (unversioned) -> 1.0.0
- Modified principles: none; initial adoption of eight principles supplied by the project.
- Added principles: Correctness First; Clean Architecture; API Contracts; Persistence & Transactions;
  Testing; Security & Observability; Maintainability; Definition of Done.
- Added sections: Decision Priority; Development Workflow.
- Removed sections: none; template placeholders replaced.
- Templates requiring updates:
  - ✅ updated: .specify/templates/plan-template.md
  - ✅ updated: .specify/templates/spec-template.md
  - ✅ updated: .specify/templates/tasks-template.md
- Follow-up TODOs: none.
-->
# Good Thing Jar Backend Constitution

## Core Principles

### I. Correctness First

Changes MUST preserve business correctness and data integrity. Implement the smallest change that
satisfies the specification. Existing behavior MUST remain unchanged unless the specification
explicitly requires a change. This limits regressions and protects established business rules.

### II. Clean Architecture

API, business, and persistence concerns MUST remain separate. Controllers MUST delegate business
decisions to the business layer, and persistence concerns MUST stay out of the API layer. Designs
SHOULD favor simple, loosely coupled components and composition; inheritance or abstraction MUST
have a concrete need. These boundaries keep behavior understandable and independently testable.

### III. API Contracts

Public APIs MUST use appropriate request and response DTOs and MUST NOT expose persistence entities.
External input MUST be validated, and errors MUST follow a consistent, centralized handling policy.
Existing API contracts MUST remain backward compatible unless a breaking change is explicitly
requested. Stable contracts protect clients from internal persistence changes.

### IV. Persistence & Transactions

Database access, JPA relationships, fetch strategies, and transaction boundaries MUST be deliberate.
Changes MUST avoid N+1 queries, unnecessary database access, and correctness that depends on an
accidental persistence-context state. Transactions and concurrent operations MUST protect data
integrity. Database schema changes MUST use version-controlled migrations. These rules make data
behavior predictable under load and concurrent use.

### V. Testing

Business-critical behavior and bug fixes MUST have appropriate automated tests. Use unit tests for
isolated logic and integration tests when Spring, JPA, database, transaction, or API behavior matters.
Tests MUST verify observable behavior rather than implementation details. This gives evidence that
the required behavior works and guards against regression.

### VI. Security & Observability

Secrets and sensitive information MUST NOT be hardcoded or exposed. Authentication, authorization,
input validation, and logging MUST be applied where the behavior requires them. Failures MUST be
diagnosable without leaking sensitive data. These safeguards protect users and support production
incident investigation.

### VII. Maintainability

Code MUST favor readability and simplicity over cleverness or premature abstraction. Follow existing
project conventions before adding patterns or dependencies. Changes MUST avoid unrelated refactoring.
Comments and documentation SHOULD explain non-obvious decisions and reasoning rather than restate
code. Focused, conventional changes are easier to review and maintain.

### VIII. Definition of Done

A change is complete only when the specification is satisfied, relevant tests pass, data integrity
and security are preserved, persistence and transaction implications have been considered, and no
unintended behavior is introduced. Reviewers MUST verify these conditions before accepting a change.

## Decision Priority

When principles conflict, resolve the tradeoff in this order:
Correctness & Data Integrity > Security > Maintainability > Testability > Performance > Convenience.
A lower-priority benefit MUST NOT override a higher-priority obligation without an explicit
amendment to this constitution.

## Development Workflow

Specifications MUST identify intended behavior and any API, validation, compatibility, data, or
security effects. Plans MUST assess architecture, DTO boundaries, persistence access, transactions,
migrations, test scope, and observability for affected areas. Tasks MUST include the implementation
and verification work needed to meet the Definition of Done. Reviews MUST check the stated contract
against the actual change and test results; unresolved risks MUST be documented before acceptance.

## Governance

This constitution governs project specifications, plans, tasks, implementation, and review. Amendments
MUST be documented in this file with a reason, impact on dependent templates, and a version change.
The version follows semantic versioning: MAJOR for incompatible principle or governance changes,
MINOR for new principles or materially expanded guidance, and PATCH for non-semantic clarification.
The ratification date remains the initial adoption date; the last-amended date changes on every
amendment. Every change review MUST check compliance with this constitution, and exceptions MUST be
explicitly justified and approved as part of the change review.

**Version**: 1.0.0 | **Ratified**: 2026-09-18 | **Last Amended**: 2026-09-18