# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See
`.agents/skills/speckit-plan/SKILL.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Affected Applications**: [backend, frontend, or both]
**Language/Version**: [per affected application or NEEDS CLARIFICATION]
**Primary Dependencies**: [backend and/or frontend dependencies or NEEDS CLARIFICATION]
**Storage**: [database and migration tool, if applicable, or N/A]
**Testing**: [applicable unit, component, integration, and browser tools or NEEDS CLARIFICATION]
**Target Platform**: [backend deployment and/or desktop and mobile browsers or NEEDS CLARIFICATION]
**Project Type**: [Spring Boot backend, browser frontend, or both]
**Performance Goals**: [domain-specific, e.g., request throughput or p95 latency, or NEEDS CLARIFICATION]
**Constraints**: [domain-specific, e.g., database limits or memory budget, or NEEDS CLARIFICATION]
**Scale/Scope**: [domain-specific, e.g., users, records, or endpoints, or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Document how the design meets each applicable gate. Mark non-applicable gates N/A with a reason.

- **Correctness and compatibility**: State the smallest behavior change and any existing behavior or
  API contract that must be preserved.
- **Repository boundaries (shared)**: Keep specs and contracts in `good-thing-jar-spec`, backend
  implementation in `../good-thing-jar-backend`, and frontend implementation in
  `../good-thing-jar-front-end`. Name the working directory for each implementation command.
- **Architecture and API (backend only)**: Identify API DTOs, validation, centralized error handling,
  business-layer decisions, and persistence-layer responsibilities.
- **Persistence and transactions (backend only)**: Describe data access, JPA relationships and fetch
  strategy, transaction boundaries, concurrency protections, and version-controlled migrations.
- **Security and observability**: Identify authentication, authorization, sensitive-data handling,
  and logging needed for the feature. Frontend logs, analytics, browser storage, and persistent
  caches must not expose tokens, entry content, or other sensitive data. Describe logout and
  session-termination cleanup of private state and query caches, including pending-request races.
- **Frontend architecture**: Separate presentation, application flows, and API access. Prefer simple
  feature-based structure; justify any alternative. Follow documented API and error contracts.
- **Backend authority (shared)**: Confirm backend authorization and lock decisions remain
  authoritative. Browser clocks and countdowns are display-only; obtain backend confirmation
  before showing unlock or protected content and handle stale state and denial safely.
- **Frontend usability**: Cover responsive desktop and mobile layouts, full desktop-browser
  prototype journeys, keyboard navigation, accessible form labels, visible focus, and meaningful
  loading, empty, error, and submission states.
- **Testing and completion**: Name behavior-focused unit and integration tests, especially for
  business-critical behavior and bug fixes. Frontend coverage must include authentication, privacy,
  session cleanup, and unlock boundaries, including browser clock changes. Record commands for
  frontend type checking, linting, production build, and relevant automated tests, plus scenarios
  and the real backend environment for integration verification. Mocks alone cannot pass this gate.
- **Maintainability**: Confirm existing conventions and justify any new abstraction or dependency.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (separate sibling application directories)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Use actual package and module paths and include only affected applications.
  Keep backend API, business, and persistence concerns separate. Keep frontend presentation,
  application flows, and API access separate. Do not place implementation under the spec root.
-->

```text
../good-thing-jar-backend/
└── src/
    ├── main/
    │   ├── java/com/goodthingjar/[feature]/
    │   │   ├── api/             # Controllers, DTOs, error handling
    │   │   ├── service/         # Business logic and transactions
    │   │   └── persistence/     # Entities and repositories
    │   └── resources/           # Configuration and versioned migrations
    └── test/java/com/goodthingjar/ # Unit, API, and integration tests

../good-thing-jar-front-end/
├── src/
│   ├── features/[feature]/
│   │   ├── presentation/       # Components and interaction states
│   │   ├── application/        # User flows and private state
│   │   └── api/                # Feature API access
│   └── shared/                 # Only shared concerns with a concrete need
└── [test-directory]/           # Behavior and browser tests per selected tooling
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
