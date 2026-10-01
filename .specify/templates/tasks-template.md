---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Include automated tests for business-critical behavior and bug fixes. Add unit tests for
isolated logic and integration tests where API, Spring, JPA, database, or transaction behavior matters.
Tests must verify behavior, and relevant tests must pass before a story is complete.
Frontend tests MUST cover business-critical authentication, privacy, session cleanup, and unlock
boundaries, including skewed browser clocks and countdown expiry without backend permission.
Frontend completion MUST include type checking, linting, a production build, relevant automated
tests, and affected-flow verification against the real backend.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions
- Qualify implementation paths by application directory and name each command's working directory

## Path Conventions

- **Specifications and contracts**: `specs/` in `good-thing-jar-spec`
- **Spring Boot backend**: `../good-thing-jar-backend/src/main/java/com/goodthingjar/`,
  `../good-thing-jar-backend/src/main/resources/`, `../good-thing-jar-backend/src/test/java/com/goodthingjar/`
- **Frontend**: `../good-thing-jar-front-end/src/features/` and test paths selected by the plan;
  separate presentation, application flows, and API access
- **Multi-module backend**: use module-prefixed equivalents of those paths
- Backend sample paths below are relative to `../good-thing-jar-backend`; generated tasks MUST
  qualify them with that sibling directory. Frontend paths MUST identify its separate sibling root.
- Execute build/run/test/migration/source-generation commands from the affected application root

<!-- 
  ============================================================================
  IMPORTANT: The tasks below are SAMPLE TASKS for illustration purposes only.
  
  The /speckit.tasks command MUST replace these with actual tasks based on:
  - User stories from spec.md (with their priorities P1, P2, P3...)
  - Feature requirements from plan.md
  - Entities from data-model.md
  - Endpoints from contracts/
  - Constitution requirements for DTOs, validation, error handling, transactions,
    migrations, security, logging, and behavior-focused tests where applicable
  - Frontend requirements for architecture, backend authority, privacy, session/cache cleanup,
    responsive accessibility, interaction states, and all frontend completion gates
  
  Tasks MUST be organized by user story so each story can be:
  - Implemented independently
  - Tested independently
  - Delivered as an MVP increment
  
  DO NOT keep these sample tasks in the generated tasks.md file.
  ============================================================================
-->

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create project structure per implementation plan
- [ ] T002 Initialize affected application projects and dependencies in their sibling directories
- [ ] T003 [P] Configure linting and formatting tools

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

Examples of foundational tasks (adjust based on your project):

Backend examples below apply only to affected backend work. For frontend work, generate foundational
tasks for API access, session lifecycle and private-query cleanup, and the chosen test/build tooling
inside `../good-thing-jar-front-end`. Do not generate backend persistence tasks for frontend-only work.

- [ ] T004 Setup database schema and migrations framework
- [ ] T005 [P] Implement authentication/authorization framework
- [ ] T006 [P] Setup API routing and middleware structure
- [ ] T007 Create base models/entities that all stories depend on
- [ ] T008 Configure error handling and logging infrastructure
- [ ] T009 Setup environment configuration management

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 1

> Select unit, API, and integration tests according to the behavior and framework boundaries involved.
> For frontend stories, select component/browser tests for critical authentication, privacy, and
> unlock boundaries. Include real-backend journey verification and desktop/mobile keyboard use.

- [ ] T010 [P] [US1] API contract test for [endpoint] in src/test/java/[package]/api/[Name]ApiTest.java
- [ ] T011 [P] [US1] Integration test for [user journey] in src/test/java/[package]/[Name]IntegrationTest.java

### Implementation for User Story 1

- [ ] T012 [P] [US1] Create [Entity1] entity in src/main/java/[package]/persistence/[Entity1].java
- [ ] T013 [P] [US1] Create [Entity2] entity in src/main/java/[package]/persistence/[Entity2].java
- [ ] T014 [US1] Implement [Service] in src/main/java/[package]/service/[Service].java (depends on T012, T013)
- [ ] T015 [US1] Implement request/response DTOs and [endpoint] in src/main/java/[package]/api/[Name]Request.java, [Name]Response.java, and [Name]Controller.java
- [ ] T016 [US1] Add validation and error handling
- [ ] T017 [US1] Add logging for user story 1 operations

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 2

- [ ] T018 [P] [US2] API contract test for [endpoint] in src/test/java/[package]/api/[Name]ApiTest.java
- [ ] T019 [P] [US2] Integration test for [user journey] in src/test/java/[package]/[Name]IntegrationTest.java

### Implementation for User Story 2

- [ ] T020 [P] [US2] Create [Entity] entity in src/main/java/[package]/persistence/[Entity].java
- [ ] T021 [US2] Implement [Service] in src/main/java/[package]/service/[Service].java
- [ ] T022 [US2] Implement request/response DTOs and [endpoint] in src/main/java/[package]/api/[Name]Request.java, [Name]Response.java, and [Name]Controller.java
- [ ] T023 [US2] Integrate with User Story 1 components (if needed)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 3

- [ ] T024 [P] [US3] API contract test for [endpoint] in src/test/java/[package]/api/[Name]ApiTest.java
- [ ] T025 [P] [US3] Integration test for [user journey] in src/test/java/[package]/[Name]IntegrationTest.java

### Implementation for User Story 3

- [ ] T026 [P] [US3] Create [Entity] entity in src/main/java/[package]/persistence/[Entity].java
- [ ] T027 [US3] Implement [Service] in src/main/java/[package]/service/[Service].java
- [ ] T028 [US3] Implement request/response DTOs and [endpoint] in src/main/java/[package]/api/[Name]Request.java, [Name]Response.java, and [Name]Controller.java

**Checkpoint**: All user stories should now be independently functional

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] TXXX [P] Documentation updates in docs/
- [ ] TXXX Focused cleanup required by the feature
- [ ] TXXX Address measured query or latency risks
- [ ] TXXX [P] Additional behavior-focused tests in src/test/java/[package]/
- [ ] TXXX Verify applicable authentication, authorization, and sensitive-data handling
- [ ] TXXX Run quickstart.md validation

For affected frontend work, generate explicit final verification tasks (with exact paths and
commands from the plan) in addition to applicable tasks above:

- [ ] TXXX Verify private state and query caches clear on logout/session termination and delayed responses cannot restore them in ../good-thing-jar-front-end/[session-test-path]
- [ ] TXXX Verify tokens, entry content, and sensitive data are absent from logs, analytics, and persistent caches in ../good-thing-jar-front-end/[privacy-test-path]
- [ ] TXXX Verify responsive desktop/mobile layouts, full desktop prototype journeys, keyboard navigation, labels, and loading/empty/error/submission states in ../good-thing-jar-front-end/[browser-test-path]
- [ ] TXXX Run type checking, linting, production build, and relevant automated tests from ../good-thing-jar-front-end; record commands and results in specs/[###-feature]/quickstart.md
- [ ] TXXX Verify affected frontend journeys against the real backend, including authentication, denied access, and immediately-before/exact-unlock behavior; record environment, scenarios, and results in specs/[###-feature]/quickstart.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable

### Within Each User Story

- Identify required tests before implementation and run them before completion
- Add version-controlled migration tasks with exact paths for schema changes
- Cover validation, centralized errors, authorization, and logging where applicable
- Review fetch strategies, query count, transaction boundaries, and concurrency where applicable
- For frontend work, separate presentation, application flows, and API access; implement private
  state/query cleanup and display-only countdowns before protected journeys depend on them
- Verify responsive accessibility and meaningful interaction states for each affected frontend story
- Pass frontend type checking, linting, production build, automated tests, and real-backend
  verification before claiming frontend completion; mocks alone cannot satisfy the backend gate
- Entities and migrations before dependent persistence services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Entities within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch independent tests for User Story 1 together:
Task: "API contract test for [endpoint] in src/test/java/[package]/api/[Name]ApiTest.java"
Task: "Integration test for [user journey] in src/test/java/[package]/[Name]IntegrationTest.java"

# Launch all models for User Story 1 together:
Task: "Create [Entity1] entity in src/main/java/[package]/persistence/[Entity1].java"
Task: "Create [Entity2] entity in src/main/java/[package]/persistence/[Entity2].java"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Run relevant tests and verify required behavior before completion
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
