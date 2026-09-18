# good-thing-jar-spec Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-09-21

## Active Technologies

- Java 21 LTS + Spring Boot 4.1.1, Spring Web MVC, Spring Security 7.1.x, Spring Data JPA 4.1.x, Spring Modulith 2.1.1, Flyway, PostgreSQL 18 (001-shared-locked-jar)
- Maven coordinates: `com.goodthingjar:good-thing-jar-backend`
- Base package: `com.goodthingjar`

## Repository Boundaries

- Specification artifacts: `C:\workspace\good-thing-jar\good-thing-jar-spec`
- Spring Boot implementation: `C:\workspace\good-thing-jar\good-thing-jar-backend`
- Java source root: `src/main/java/com/goodthingjar/`
- Test source root: `src/test/java/com/goodthingjar/`

Keep `specs/`, planning documents, and contracts in the specification repository. Run Maven,
Flyway-backed application startup, tests, Docker Compose, and source generation from the
implementation repository.

## Backend Project Structure

```text
C:\workspace\good-thing-jar\good-thing-jar-backend\
├── pom.xml
├── mvnw
├── mvnw.cmd
├── .mvn/
├── compose.yaml                                      # planned; not yet present
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
    │   └── db/migration/                             # planned
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

## Code Style

Java 21 LTS: Follow standard conventions

## Recent Changes

- 001-shared-locked-jar: Planned a modular Spring Boot REST backend with PostgreSQL and Flyway in
  the sibling `good-thing-jar-backend` repository

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
