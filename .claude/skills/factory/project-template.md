# Factory project config

Facts the factory needs about this repo. Every factory agent reads this file; the orchestrator reads it for reviewer selection and merge. Keep each section short and concrete. Write "none" where a section does not apply.

## Context
Docs every agent reads first (usually `CLAUDE.md`), and docs only the spec-writer also reads (product plan, scope).

## Branch and push
Main branch name. Whether to push after a merge, and the exact command (normally `git push origin <main>`). What a push triggers (deploy, CI).

## Checks
Working directory for commands. Install command for a fresh worktree. Typecheck, lint, unit test and build commands. Where unit tests live and their file pattern. Database or integration tests: the command, what they need, and when to run them.

## Toolchain
One line per tool the Checks and App sections need: `name | check command | start command | needed for`. The start command only for services that can be started (Docker, a local database); "none" for plain tools. Example: `docker | docker info | open -a Docker | database tests`, `java | java -version | none | gradle build`.

## Conventions
Rules the code reviewer checks beyond "match the surrounding code": data rules, framework rules, required tests for schema changes, comment policy.

## Reviewer selection
Path prefixes that trigger `review-ux` (UI code, styles). Path prefixes that trigger `review-security` (data access, IO, auth, schema, dependency manifests and lock files). `review-code` always runs.

## Escalate
Project paths behind the approver's escalation categories: schema and migration paths, access-control policies, dependency manifests and lock files, build and infra config.

## Security checklist
What the security reviewer looks for in this app: untrusted input sources, data access and authorization model, key handling, delete rules.

## UX checklist
Primary user and their main task. Product stance (what the app must not do). Design system, where the tokens live, and the shared components new UI should reuse. Viewports to check, and whether dark mode exists.

## App in the browser
"none" if there is no UI. Otherwise: serve command (it takes `--port <port>`), how an agent signs in (signup flow, throwaway account), where the data lives (shared or local), and quirks (slow fonts, seed data needed first).
