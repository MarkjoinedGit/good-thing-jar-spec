# good-thing-jar-spec Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-10-01

## Active Technologies

- Strict TypeScript, React, Node.js 24 LTS, npm, Vite, React Router, TanStack Query,
  CSS Modules, @js-temporal/polyfill; ESLint, Vitest, React Testing Library, MSW,
  Playwright and test-process-only pg (002-responsive-web-frontend; planned)
- Java 21 LTS + Spring Boot 4.1.1, Spring Web MVC, Spring Security 7.1.x, Spring Data JPA 4.1.x, Spring Modulith 2.1.1, Flyway, PostgreSQL 18 (001-shared-locked-jar)
- Maven coordinates: `com.goodthingjar:good-thing-jar-backend`
- Base package: `com.goodthingjar`

## Repository Boundaries

- Specification artifacts: `C:\workspace\good-thing-jar\good-thing-jar-spec`
- Spring Boot implementation: `C:\workspace\good-thing-jar\good-thing-jar-backend`
- Frontend implementation: `C:\workspace\good-thing-jar\good-thing-jar-front-end`
- Java source root: `src/main/java/com/goodthingjar/`
- Test source root: `src/test/java/com/goodthingjar/`

Keep `specs/`, planning documents, and contracts in the specification repository. Run Maven,
Flyway-backed application startup, backend tests, Docker Compose, and backend source generation from
the backend repository. Run frontend startup, type checking, linting, production builds, tests, and
source generation from the frontend repository, using commands chosen by its implementation plan.

## Backend Project Structure

```text
C:\workspace\good-thing-jar\good-thing-jar-backend\
├── pom.xml
├── mvnw
├── mvnw.cmd
├── .mvn/
├── compose.yaml                                      # PostgreSQL and Mailpit
└── src/
    ├── main/java/com/goodthingjar/
    │   ├── GoodThingJarBackendApplication.java
    │   ├── identity/
    │   ├── pairing/
    │   ├── jar/
    │   ├── notification/
    │   └── platform/
    ├── main/resources/
    │   ├── application.properties
    │   └── db/migration/
    └── test/java/com/goodthingjar/
```

The application class and feature modules use the authoritative `com.goodthingjar` base package.

## Commands

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-backend
.\mvnw.cmd spring-boot:run
.\mvnw.cmd test
.\mvnw.cmd package
.\mvnw.cmd verify
```

Planned frontend commands become available after frontend bootstrap:

```powershell
Set-Location -LiteralPath C:\workspace\good-thing-jar\good-thing-jar-front-end
npm ci
npm run dev
npm run typecheck
npm run lint
npm run build
npm test
npm run test:e2e
npm run test:e2e:backend
```

Record compatible installed versions and commit `package-lock.json` in the frontend repository.
Use the `/api/v1` Vite proxy without rewriting the backend prefix. See
`specs/002-responsive-web-frontend/quickstart.md` for isolated real-backend test setup and gates.
No implementation commands run from the specification root.

## Code Style

Java 21 LTS: Follow standard conventions

## Recent Changes

- 002-responsive-web-frontend: Planned a responsive Vietnamese frontend in the sibling
  `good-thing-jar-front-end` repository with coordinated renewal, memory-only private state,
  explicit mutation recovery, responsive accessibility and real-backend verification.
- 001-shared-locked-jar: Planned a modular Spring Boot REST backend with PostgreSQL and Flyway in
  the sibling `good-thing-jar-backend` repository

<!-- MANUAL ADDITIONS START -->
## Constitution Compliance

Follow `.specify/memory/constitution.md` (v1.1.0) for both applications. Backend controller/business/
persistence separation and database/transaction/migration principles apply only to the backend.
Shared business rules, security requirements, and API compatibility obligations apply to both apps.

Frontend work MUST separate presentation, application flows, and API access, preferably by feature.
The backend is authoritative for authorization and jar lock status; browser clocks and countdowns
are display-only. Tokens, entry content, and sensitive data MUST NOT appear in logs, analytics, or
persistent caches. Logout and session termination MUST clear private state and query caches, and
delayed requests MUST NOT restore that state.

Frontend flows MUST support responsive desktop/mobile layouts, keyboard navigation, accessible
labels, and meaningful loading, empty, error, and submission states. The initial prototype MUST be
fully usable and testable in a desktop browser. Test business-critical authentication, privacy, and
unlock boundaries. Frontend completion requires passing type checking, linting, a production build,
relevant automated tests, and verification against the real backend, with results recorded.
<!-- MANUAL ADDITIONS END -->
