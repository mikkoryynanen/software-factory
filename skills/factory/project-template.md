# Factory project config

Facts the factory needs about this repo. Every factory agent reads this file; the orchestrator reads it for reviewer selection and merge. Keep each section short and concrete. Write "none" where a section does not apply.

## Context
Docs every agent reads first (usually `CLAUDE.md`), and docs only the spec-writer also reads (product plan, scope).

## Sources
Knowledge sources outside the repo, one source scout each per job. One line per source: `name | access | what it holds`. Access is a folder path (read with Read and Grep; add it to `permissions.additionalDirectories`) or the MCP server whose tools search it (allow only its search and read tools; scouts never write). Example: `obsidian | ~/Notes/Work | meeting notes, decisions`, `jira | MCP atlassian | tickets for project CRON`. "none" if there are none; the code scout always runs.

## Branch and push
Main branch name. `Delivery: merge` (an approved job is merged into main locally) or `Delivery: pr` (the factory pushes `factory/<job-id>` to `origin` and opens a PR with `gh`; you merge it on GitHub). For `merge`: whether to push main after a merge, and the exact command (normally `git push origin <main>`). What a push triggers (deploy, CI).

## Checks
Working directory for commands. Install command for a fresh worktree. Typecheck, lint, unit test and build commands. Where unit tests live and their file pattern. Database or integration tests: the command, what they need, and when to run them.

## Toolchain
One line per tool the Checks and App sections need: `name | check command | start command | needed for`. The start command only for services that can be started (Docker, a local database); "none" for plain tools. Example: `docker | docker info | open -a Docker | database tests`, `java | java -version | none | gradle build`.

## Conventions
Rules the code reviewer checks beyond "match the surrounding code": data rules, framework rules, required tests for schema changes, comment policy.

## Reviewer selection
Path prefixes that trigger `review-ux` (UI code, styles). `review-code` and `review-security` always run.

## Escalate
Project paths behind the approver's escalation categories: schema and migration paths, access-control policies, dependency manifests and lock files, build and infra config.

## Security checklist
What the security reviewer looks for in this app: untrusted input sources, data access and authorization model, key handling, delete rules. Accepted risks: known gaps the owner accepts (for example "admin revocation takes up to 15 minutes"), so reviewers do not re-raise them.

## UX checklist
Primary user and their main task. Product stance (what the app must not do). Design system, where the tokens live, and the shared components new UI should reuse. Viewports to check, and whether dark mode exists.

## Data
Whether the database holds real data that migrations must keep. Throwaway DB command: how to get a disposable copy of the database (or back it up) before testing a schema change. Real-config run: how to run the app with the real env (for example a real identity provider), and who does it (agents cannot read env files).

## App in the browser
"none" if there is no UI. Otherwise: serve command (it takes `--port <port>`), how an agent signs in (signup flow, throwaway account), where the data lives (shared or local), and quirks (slow fonts, seed data needed first).
