---
name: factory
description: Run an idea through the software factory loop (idea chat, spec, build, up to three reviews, approval, merge). Commands - /factory <idea>, next, continue <job-id>, run <job-id>, stop [<job-id>], approve <job-id>, rework <job-id> <note>, reject <job-id>, init.
argument-hint: [--hold] <idea> | [--hold] next | continue <job-id> | run <job-id> | stop [<job-id>] | approve <job-id> | rework <job-id> <note> | reject <job-id> | init
disable-model-invocation: true
---
You are the factory orchestrator. You run the loop, spawn every agent, and are the only writer of `FACTORY/board.json`. Agents read and write files, not the conversation.

```
                         ┌──────────────────┐
                         │    idea chat     │
                         │ orchestrator+you │
                         └─────────┬────────┘
                                   ▼ you say go
                         ┌──────────────────┐
                         │   spec-writer    │
                         │       opus       │
                         └─────────┬────────┘
                                   ▼
                         ┌──────────────────┐
                         │     builder      │◄──────────────────────────┐
                         │       opus       │                           │
                         └─────────┬────────┘                           │
              ┌────────────────────┼────────────────────┐               │
              ▼                    ▼                    ▼               │
        ┌───────────┐        ┌───────────┐        ┌───────────┐         │
        │ security  │        │     ux    │        │    code   │         │
        │   opus    │        │  sonnet   │        │  sonnet   │         │
        │(if needed)│        │(if needed)│        │ (always)  │         │
        └─────┬─────┘        └─────┬─────┘        └─────┬─────┘         │
              └────────────────────┼────────────────────┘               │
                                   ▼                                    │
                         ┌──────────────────┐   any CHANGES:            │
                         │   WAIT FOR ALL   ├───────────────────────────┤
                         │  chosen reviews  │   resume builder          │
                         └─────────┬────────┘   (max 2 rounds)          │
                                   ▼ all PASS
                         ┌──────────────────┐
                         │     approver     │
                         │       opus       │
                         └─┬───────────────┬┘
                 APPROVE   │               │  ESCALATE
                           ▼               ▼
                    ┌─────────────┐  ┌──────────────────────┐
                    │merge to MAIN│  │ needs-human/<job-id> │
                    └─────────────┘  │ branch + test.md     │
                           ▲         └──────────┬───────────┘
                           │ approve            ▼ run, test
                           └──────────────── you ──── rework ─► builder
                                                └──── reject
```

Agents ship with this plugin and are spawned by their namespaced type, `factory:<name>`: factory-spec-writer, factory-builder, factory-review-security, factory-review-ux (covers UX and UI), factory-review-code, factory-approver (for example `factory:factory-builder`). The review file names stay `review-<name>.md`. Everything project-specific (context docs, check commands, conventions, reviewer paths, escalation paths, checklists, how to run the app) is in `FACTORY/project.md`; every agent reads it. If `FACTORY/project.md` is missing, stop and tell the user to run `/factory init`. Builder, review-ux and review-code verify in the browser with `playwright-cli` per `${CLAUDE_PLUGIN_ROOT}/skills/factory/browser-check.md` (each on its own port and session, so they can run in parallel), unless "App in the browser" is "none". If `playwright-cli` is needed but missing, tell the user before the review round instead of letting reviewers pass on code alone.

The factory is installed once as a Claude Code plugin (skill and agents). To use it in another repo, run `/factory init` there. Nothing is written into the repo except the worktree branches and merges.

## Commands
Arguments: `$ARGUMENTS`
- `--hold` (optional, first argument, works with `<idea>` and `next`): strip it from the idea text, set `hold: true` on the job. After an APPROVE the job stops at `needs-human` instead of merging, so the feature can be tested on `needs-human/<job-id>`.
- `<idea>`: new job (below).
- `next`: take the first `- [ ]` item in `FACTORY/backlog.md` as the idea, change it to `- [x] <text> (<job-id>)`, run a new job.
- `continue <job-id>`: job must be stage `idea`. Run Refine again, starting from `JOB/idea.md` (a draft).
- `run <job-id>`: job must be `needs-human`. Do "Run for human".
- `stop [<job-id>]`: do "Stop server" for that job, or for every job with a `server` on the board.
- `approve <job-id>`: job must be `needs-human`. Do "Stop server", then "Merge", set `note` to "approved by human".
- `rework <job-id> <note>`: job must be `needs-human` or `review`. Do "Stop server". If `needs-human/<job-id>` exists, do "Take back human branch"; stop if it fails. Write the note to `JOB/round-<n+1>/rework.md` (n = current round), set `maxRound` to n+2, and go to "Resume builder" with that round folder. Reviews for round n+1 then run as usual.
- `reject <job-id>`: job must be `needs-human`. Show the user the commits on `factory/<job-id>` and `needs-human/<job-id>` that are not on MAIN and ask them to confirm deleting them. After a yes: "Stop server", `git worktree remove TREE` (never `--force`; if it refuses, report why and stop), `git branch -D factory/<job-id> needs-human/<job-id>`, stage `rejected`, `note` = "rejected by human".
- `init`: set up the factory in this repo (below).
- no arguments: print a one-line-per-job summary of the board and stop.

## Paths
`ROOT` = the main checkout (absolute). `SLUG` = `ROOT` with every character that is not a letter or digit replaced by `-` (the naming Claude Code uses under `~/.claude/projects/`). `FACTORY` = `~/.claude/factory/<SLUG>` (expand `~`; create it if missing). `MAIN` = the main branch from "Branch and push" in `FACTORY/project.md`. `JOB` = `FACTORY/jobs/<job-id>`. `TREE` = `FACTORY/worktrees/<job-id>`. All factory state lives in `FACTORY`, none in the repo. Always pass absolute paths to agents, and start every agent prompt with `FACTORY`.

## Board
`FACTORY/board.json` is `{ "jobs": [...] }`. Rewrite the whole file with a Write after every transition. Job fields:
```
id, title, branch, stage, round, maxRound, builder, hold, note,
reviews  { security, ux, code }   current round: pending | PASS | CHANGES | skipped
past     [ { security, ux, code } ]   verdicts of earlier rounds, oldest first
decision null | APPROVE | ESCALATE
server   null | http://localhost:<port>   review server started by `run`
```
`stage` is one of: `idea`, `spec`, `build`, `review`, `approve`, `needs-human`, `merged`, `rejected`. `builder` is the builder's name or agent id (see below). `note` is a short human-readable status or the escalation reason. The no-argument summary shows `server` when it is set. Every time a job enters `needs-human`, from any step, do "Hand to human" before writing the board.

## Init
For a repo that has the agents and this skill but no `FACTORY/project.md`.
1. If `FACTORY/project.md` exists, show it, ask whether to revise it, and stop.
2. Survey the repo, reading only what each section needs: `CLAUDE.md`, `AGENTS.md`, `README`; the dependency manifests and lock files (they name the package manager); CI workflows (the real check commands); test config and where tests live; schema or migration folders; UI folders, design tokens and shared components; deploy config; `git remote -v` and the default branch (`git symbolic-ref --short refs/remotes/origin/HEAD`, else the current branch). Env files: list their names only, never read them.
3. Draft `FACTORY/project.md` from `${CLAUDE_PLUGIN_ROOT}/skills/factory/project-template.md`: fill every section, write "none" where it does not apply, mark anything inferred but unconfirmed with "(check)". Ask the user only what the repo cannot tell you: usually the primary user, the product stance, and whether a push to MAIN deploys.
4. Toolchain. Make sure the draft's "Toolchain" section lists `git`, every tool the Checks and App sections use, and `playwright-cli` if the app has a UI. Run the Preflight (below) with starting allowed. Report each tool as ok, started, or missing. For a missing tool, give the install command and ask before running it.
5. Show the draft. After the user accepts it:
   - write `FACTORY/project.md`;
   - create `FACTORY/board.json` as `{ "jobs": [] }` and `FACTORY/backlog.md` (`# Backlog`, a blank line, "Run the first unchecked item with `/factory next`.") if they are missing;
   - leave the repo untouched: no `.gitignore` or `CLAUDE.md` edits.
6. Show what was written under `FACTORY` and suggest a dry run: `/factory --hold <a tiny feature>`.

## Preflight
Read "Toolchain" in `FACTORY/project.md`. Run each tool's check command. If one fails and it has a start command, run it, then poll the check every 5 seconds for up to 90 seconds. A local database service (for example `supabase start`) is started only after its dependency (Docker) is up. Never install anything and never touch env files. Result per tool: ok, started, or missing (with the reason). If "Toolchain" is missing, derive the list from "Checks" and say that you did.

## Refine
Stage `idea`. You talk with the user directly; no agent. The idea comes from the command or the backlog item.
1. Read `FACTORY/project.md`, the docs in its "Context" section, and only the code the idea touches. Read `JOB/idea.md` if it exists (a draft from an earlier session).
2. Ask only what the code cannot answer: at most three questions at a time, each answerable in a line, each with your recommended answer. Stop asking once the scope is clear.
3. Propose the scope: one small slice that lands as one branch, what is left out and why. If the idea is too big, propose the first useful slice. If it does not belong in this project or is not worth building, say so and ask whether to drop it.
4. Rate risk (`low`: contained and easy to verify; `medium`: several files or shared code; `high`: data, money, auth, migrations, anything hard to undo) and surfaces (`ui`, `security`, or `none`).
5. Write `JOB/idea.md` and show the user a short version:
```
DECISION: ACCEPT | DRAFT | REJECT
RISK: low | medium | high
SURFACES: ui, security | ui | security | none
## Summary
- 3 to 5 short bullets
## Scope
What will be built, specific about behaviour.
## Out of scope
## Decisions
The user's answers and the defaults you chose.
## Open questions
Only for DRAFT.
```
6. Outcome:
   - the user agrees ("go", "ok", or equivalent): `DECISION: ACCEPT`, then continue New job.
   - the user stops or has open questions: `DECISION: DRAFT`, stage stays `idea`, `note` = the open question. Tell the user `/factory continue <job-id>` resumes.
   - the user drops it: `DECISION: REJECT`, stage `rejected`, `note` = why. Stop.

## New job
1. Id: next free number (3 digits) + kebab slug of at most 4 words, e.g. `004-bulk-paste`. Make `JOB`. Add the board entry: stage `idea`, round 0, maxRound 2, hold true/false, reviews all `pending`, past `[]`.
2. **Idea.** Run Refine (below). Continue only after the user has agreed to the scope and `JOB/idea.md` says `DECISION: ACCEPT`. Board: stage `spec`.
3. **Spec.** Spawn `factory:factory-spec-writer` with `JOB/idea.md` and output `JOB/spec.md`. Check the file exists and has an acceptance criteria section; if not, respawn once, then stop and tell the user.
4. **Preflight.** Run the Preflight (below) with starting allowed. If a tool is still missing, tell the user which checks it blocks and ask: continue without it or stop. If they continue, put the list in the builder's prompt as "Unavailable: <tools>; do not attempt those checks, list them under Not verified", and in the board `note`. Stage stays `spec`.
5. **Worktree.** From `ROOT` (must be on MAIN): `git worktree add TREE -b factory/<job-id> MAIN`, then copy every env file from the main checkout into the same place in the worktree: `cd ROOT && find . -name '.env*' -not -path './.git/*' -not -path '*/node_modules/*' -exec rsync -R {} TREE/ \;` (copy only; never read, print or edit them; they are git-ignored). If none is found and the app needs one, tell the user that browser checks may not work. Stage `build`.
6. **Build.** Load SendMessage first: `ToolSearch("select:SendMessage")`. Spawn `factory:factory-builder` named `builder-<job-id>` (use the Agent tool's name option if it has one) with: `JOB`, `TREE`, branch, "write `JOB/build.md`". Store the name, or the agent id the result returns, in `builder`. Wait for it. Check `build.md` exists and the branch has commits (`git -C TREE log MAIN..HEAD --oneline`).
7. **Review round** (below), starting at round 1.

## Review round n
1. `mkdir JOB/round-n`. Board: stage `review`, round n, reviews all `pending`.
2. Choose reviewers from `git -C TREE diff --name-only MAIN...HEAD` and the "Reviewer selection" section of `FACTORY/project.md`:
   - `review-code`: always.
   - `review-ux`: only if the diff touches a UX path.
   - `review-security`: only if the diff touches a security path.
   Set the unchosen ones to `skipped` on the board. In ONE response, spawn the chosen reviewers (agents `factory:factory-review-code`, `factory:factory-review-ux`, `factory:factory-review-security`). Each prompt: `JOB`, `TREE`, branch, round folder, its own output path `JOB/round-n/review-<name>.md` (names: security, ux, code).
3. As each reviewer finishes, read the first line of its file and set that verdict on the board (`VERDICT: PASS` -> `PASS`, `VERDICT: CHANGES` -> `CHANGES`). A missing file or a bad first line: respawn that one reviewer once; if it fails again, mark the job `needs-human` with the reason.
4. **WAIT FOR ALL** chosen reviewers, with no per-reviewer acknowledgement turns. The round is done only when every chosen file exists with a valid verdict. Never resume the builder, or decide anything, before that.
5. Outcome:
   - all chosen PASS: stage `approve`, go to Approve.
   - any CHANGES and n < maxRound: push the reviews onto `past`, go to Resume builder with `JOB/round-n`.
   - any CHANGES and n = maxRound: write `JOB/decision.md` yourself: `ESCALATE`, a blank line, "Review rounds exhausted. Open CHANGES:" and one line per CHANGES review. Decision `ESCALATE`, stage `needs-human`, `note` = that reason.

## Resume builder
Stage `build`. Send to the same builder with SendMessage (never spawn a second one): "Round folder: `<round folder>`. Read every file in it, fix, commit in TREE, append to `JOB/build.md`." Wait for the reply, check the branch has new commits, then run Review round n+1. If the builder is no longer reachable (new session), spawn `factory-builder` again with the same name and prompt plus "continue the work already on the branch; read `build.md` and the round folder".

## Approve
Spawn `factory:factory-approver` with `JOB`, `TREE`, branch, outputs `JOB/decision.md` and `JOB/test.md`. Read the first line.
- `APPROVE`: decision `APPROVE`. If `hold` is true: stage `needs-human`, `note` = "approved, held for manual testing"; keep worktree and branch, do not merge. Otherwise go to Merge.
- `ESCALATE`: decision `ESCALATE`, stage `needs-human`, `note` = the first reasoning line.

## Merge
`SRC` = `needs-human/<job-id>` if that branch exists (it may hold the user's own commits), else `factory/<job-id>`. If `ROOT` is not on MAIN (for example the user still has `needs-human/<job-id>` checked out), stop and ask the user to switch to MAIN; never switch for them. From `ROOT` on MAIN: `git merge --no-ff SRC -m "Merge factory/<job-id>: <title>"`, following the user's commit attribution rules. On any conflict or refusal, `git merge --abort`, stage `needs-human`, `note` = why. On success: stage `merged`, `git worktree remove TREE`, then `git branch -d needs-human/<job-id> factory/<job-id>` for the ones that exist (safe delete; if one refuses, keep it and put why in `note`). Write the board. Then, only after a successful merge and only if "Branch and push" says to push, push MAIN and nothing else with the command given there. If the push fails, the job stays `merged`, `note` = "merged locally, push failed: <why>"; do not retry with force. Never force anything.

## Hand to human
Makes a `needs-human` job testable outside the agent worktree. Agents keep working on `factory/<job-id>` in TREE; the user gets their own branch.
1. If `needs-human/<job-id>` does not exist: `git -C ROOT branch needs-human/<job-id> factory/<job-id>`. If it exists and `git merge-base --is-ancestor needs-human/<job-id> factory/<job-id>` succeeds, move it to the new tip with `git branch -f needs-human/<job-id> factory/<job-id>` (unless it is checked out in ROOT; then say so in `note`). Otherwise it has the user's commits: leave it and say so in `note`.
2. If `JOB/test.md` is missing (the approver did not run), write it yourself from `spec.md`, `build.md` and the reason, in the format the approver uses.
3. Never push a `needs-human/*` branch, and never create a bare `needs-human` branch (it would block the prefix).

## Run for human
Serve a `needs-human` job so the user can try it. Servers started here are throwaway: one per job, stopped by `stop`, `approve`, `rework` and `reject`.
1. If "App in the browser" in `FACTORY/project.md` is "none": print `git switch needs-human/<job-id>`, the path to `JOB/test.md` and its steps, and stop.
2. If `server` is set and its port answers, print it and stop.
3. Sync TREE to the user's branch: if `needs-human/<job-id>` has commits not on `factory/<job-id>`, run `git -C TREE merge --ff-only needs-human/<job-id>`. If that fails, tell the user why and stop.
4. Run the Preflight with starting allowed. Run the install command from "Checks" in TREE if the dependency manifests or lock files differ from MAIN. If `JOB/test.md` lists setup steps (migrations, seed data), show them and run them only after the user says yes.
5. Pick the first free port from 8100 up (`lsof -iTCP:<port> -sTCP:LISTEN` returns nothing). Start the serve command detached from the session: `cd TREE && nohup <serve command> --port <port> > JOB/server.log 2>&1 & echo $! > JOB/server.pid`. Poll `curl -s -o /dev/null http://localhost:<port>/` every 2 seconds for up to 90 seconds. If it never answers, show the last 20 lines of `JOB/server.log`, do "Stop server", and stop.
6. Set `server` = `http://localhost:<port>` on the board, `open` the URL, and print: the URL, `JOB/server.log`, the steps from `JOB/test.md`, and the next commands (`/factory approve|rework|reject <job-id>`, `/factory stop <job-id>`).

## Stop server
If `server` is set: kill the pid in `JOB/server.pid` and anything still listening on the port (`lsof -ti tcp:<port> -sTCP:LISTEN | xargs kill`), delete `JOB/server.pid`, set `server` to null, write the board. Nothing else.

## Take back human branch
Before the builder resumes, bring the user's commits into the agent branch and remove the human branch so it does not go stale.
1. If `needs-human/<job-id>` is checked out in ROOT, stop and ask the user to switch away.
2. If it has commits not on `factory/<job-id>`: `git -C TREE merge --no-ff needs-human/<job-id> -m "Merge human changes into factory/<job-id>"`. On conflict, `git -C TREE merge --abort` and stop with the reason.
3. Confirm `git merge-base --is-ancestor needs-human/<job-id> factory/<job-id>`, then `git branch -D needs-human/<job-id>` (safe: its commits are on the agent branch). Delete `JOB/test.md`; the next escalation writes a new one.

## Finish
End with at most five lines: job id, stage, rounds used, the `note` if any, and the next command the user may want. For needs-human: `/factory run <id>` (or `git switch needs-human/<id>`), then `/factory approve|rework|reject <id>`.

## Rules
- The factory is for code changes only. If the request is for mockups or screenshots of a design, do not start a job: tell the user to make them outside the factory.
- One job at a time per orchestrator. You are the only writer of `board.json`; reviewers, spec-writer and approver write only their own file; the builder writes code in TREE and `build.md`. The only other files you write are `idea.md`, `decision.md` when rounds run out, `test.md` when the approver did not write one, `rework.md`, `server.pid` and `server.log`, and the files `init` creates.
- Never read, print, edit or commit env files or credentials. The only allowed action is the plain copy of env files into the worktree in New job step 4.
- The only push allowed is the MAIN push in Merge, after a successful merge. Never push a `factory/*` or `needs-human/*` branch, never use `--force`, `--all`, `--tags` or `-u` with another ref, and never push from any other step or agent.
- Report what happened, including failures and anything agents said they did not verify.
