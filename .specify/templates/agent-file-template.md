# [PROJECT NAME] Development Guidelines

Auto-generated from all feature plans. Last updated: [DATE]

## Active Technologies

[EXTRACTED FROM ALL PLAN.MD FILES]

## Repository Boundaries & Constitution

Follow `.specify/memory/constitution.md` in `good-thing-jar-spec` for shared and application-specific
principles. Keep specifications, plans, tasks, and contracts in that directory. Keep backend
implementation in the sibling `good-thing-jar-backend` and frontend implementation in the separate
sibling `good-thing-jar-front-end`. Run commands from the application directory they affect.

For frontend work, separate presentation, application flows, and API access. Treat backend
authorization and jar lock status as authoritative; clocks/countdowns are display-only. Protect
tokens and entry content from logs, analytics, and persistent caches, and clear private state and
query caches on logout/session termination. Verify responsive desktop/mobile use, full desktop
prototype journeys, keyboard navigation, labels, and interaction states. Frontend completion requires
type checking, linting, a production build, relevant automated tests, and real-backend verification.

## Project Structure

```text
[ACTUAL STRUCTURE FROM PLANS]
```

## Commands

[ONLY COMMANDS FOR ACTIVE TECHNOLOGIES]

## Code Style

[LANGUAGE-SPECIFIC, ONLY FOR LANGUAGES IN USE]

## Recent Changes

[LAST 3 FEATURES AND WHAT THEY ADDED]

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
