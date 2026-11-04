---
name: factory
description: Run a feature through the software factory loop (spec, build, up to three reviews, approval, merge). Commands - /factory <feature>, next, approve <job-id>, rework <job-id> <note>, init.
argument-hint: [--hold] <feature> | [--hold] next | approve <job-id> | rework <job-id> <note> | init
disable-model-invocation: true
---
You are the factory orchestrator. You run the loop, spawn every agent, and are the only writer of `factory/board.json`. Agents read and write files, not the conversation.

```
feature -> spec-writer -> builder -> [security | ux | code] -> all PASS -> approver -> APPROVE: merge to MAIN
                             ^                  | any CHANGES (max 2 rounds)  \-> ESCALATE: needs-human
                             \------ resume ----/
```

Agents are in `.claude/agents/`: spec-writer, builder, review-security, review-ux (covers UX and UI), review-code, approver. Everything project-specific (context docs, check commands, conventions, reviewer paths, escalation paths, checklists, how to run the app) is in `factory/project.md`; every agent reads it. If `factory/project.md` is missing, stop and tell the user to run `/factory init`. Builder, review-ux and review-code verify in the browser with `playwright-cli` per `.claude/skills/factory/browser-check.md` (each on its own port and session, so they can run in parallel), unless "App in the browser" is "none". If `playwright-cli` is needed but missing, tell the user before the review round instead of letting reviewers pass on code alone.

To bring the factory to another repo: copy `.claude/agents/` (the six factory agents) and `.claude/skills/factory/`, then run `/factory init` there.

## Commands
Arguments: `$ARGUMENTS`
- `--hold` (optional, first argument, works with `<feature>` and `next`): strip it from the feature text, set `hold: true` on the job. After an APPROVE the job stops at `needs-human` instead of merging, so the feature can be tested in TREE.
- `<feature>`: new job (below).
- `next`: take the first `- [ ]` item in `factory/backlog.md` as the feature, change it to `- [x] <text> (<job-id>)`, run a new job.
- `approve <job-id>`: job must be `needs-human`. Do "Merge", set `note` to "approved by human".
- `rework <job-id> <note>`: job must be `needs-human` or `review`. Write the note to `factory/jobs/<job-id>/round-<n+1>/rework.md` (n = current round), set `maxRound` to n+2, and go to "Resume builder" with that round folder. Reviews for round n+1 then run as usual.
- `init`: set up the factory in this repo (below).
- no arguments: print a one-line-per-job summary of the board and stop.

## Paths
`ROOT` = the main checkout (absolute). `MAIN` = the main branch from "Branch and push" in `factory/project.md`. `JOB` = `ROOT/factory/jobs/<job-id>`. `TREE` = `ROOT/.claude/worktrees/<job-id>` (git-ignored). Always pass absolute paths to agents.

## Board
`factory/board.json` is `{ "jobs": [...] }`. Rewrite the whole file with a Write after every transition. Job fields:
```
id, title, branch, stage, round, maxRound, builder, hold, note,
reviews  { security, ux, code }   current round: pending | PASS | CHANGES | skipped
past     [ { security, ux, code } ]   verdicts of earlier rounds, oldest first
decision null | APPROVE | ESCALATE
```
`stage` is one of: `spec`, `build`, `review`, `approve`, `needs-human`, `merged`. `builder` is the builder's name or agent id (see below). `note` is a short human-readable status or the escalation reason.

## Init
For a repo that has the agents and this skill but no `factory/project.md`.
1. If `factory/project.md` exists, show it, ask whether to revise it, and stop.
2. Survey the repo, reading only what each section needs: `CLAUDE.md`, `AGENTS.md`, `README`; the dependency manifests and lock files (they name the package manager); CI workflows (the real check commands); test config and where tests live; schema or migration folders; UI folders, design tokens and shared components; deploy config; `git remote -v` and the default branch (`git symbolic-ref --short refs/remotes/origin/HEAD`, else the current branch). Env files: list their names only, never read them.
3. Draft `factory/project.md` from `.claude/skills/factory/project-template.md`: fill every section, write "none" where it does not apply, mark anything inferred but unconfirmed with "(check)". Ask the user only what the repo cannot tell you: usually the primary user, the product stance, and whether a push to MAIN deploys.
4. Check the tools the draft names: `git`, each check command's tool (`--version`), and `playwright-cli` if the app has a UI. Report any that are missing.
5. Show the draft. After the user accepts it:
   - write `factory/project.md`;
   - create `factory/board.json` as `{ "jobs": [] }` and `factory/backlog.md` (`# Backlog`, a blank line, "Run the first unchecked item with `/factory next`.") if they are missing;
   - add `.claude/worktrees/` and `factory/jobs/*/screens/` to `.gitignore` if absent;
   - add a short "Software factory" section to `CLAUDE.md` (create the file if there is none): `/factory` runs features through spec, build, reviews and an approver; config in `factory/project.md`, state in `factory/`; builders commit on local `factory/<job-id>` branches in worktrees, which are never pushed; only an APPROVE or `/factory approve` merges to MAIN and then pushes MAIN only.
6. Do not commit. Show what changed and suggest a dry run: `/factory --hold <a tiny feature>`.

## New job
1. Id: next free number (3 digits) + kebab slug of at most 4 words, e.g. `004-bulk-paste`. Make `JOB`. Add the board entry: stage `spec`, round 0, maxRound 2, hold true/false, reviews all `pending`, past `[]`.
2. **Spec.** Spawn `spec-writer` with the feature and output `JOB/spec.md`. Check the file exists and has an acceptance criteria section; if not, respawn once, then stop and tell the user.
3. **Worktree.** From `ROOT` (must be on MAIN): `git worktree add TREE -b factory/<job-id> MAIN`, then copy every env file from the main checkout into the same place in the worktree: `cd ROOT && find . -name '.env*' -not -path './.git/*' -not -path '*/node_modules/*' -not -path './.claude/worktrees/*' -exec cp --parents {} TREE \;` (copy only; never read, print or edit them; they are git-ignored). If none is found and the app needs one, tell the user that browser checks may not work. Stage `build`.
4. **Build.** Load SendMessage first: `ToolSearch("select:SendMessage")`. Spawn `builder` named `builder-<job-id>` (use the Agent tool's name option if it has one) with: `JOB`, `TREE`, branch, "write `JOB/build.md`". Store the name, or the agent id the result returns, in `builder`. Wait for it. Check `build.md` exists and the branch has commits (`git -C TREE log MAIN..HEAD --oneline`).
5. **Review round** (below), starting at round 1.

## Review round n
1. `mkdir JOB/round-n`. Board: stage `review`, round n, reviews all `pending`.
2. Choose reviewers from `git -C TREE diff --name-only MAIN...HEAD` and the "Reviewer selection" section of `factory/project.md`:
   - `review-code`: always.
   - `review-ux`: only if the diff touches a UX path.
   - `review-security`: only if the diff touches a security path.
   Set the unchosen ones to `skipped` on the board. In ONE response, spawn the chosen reviewers. Each prompt: `JOB`, `TREE`, branch, round folder, its own output path `JOB/round-n/review-<name>.md` (names: security, ux, code).
3. As each reviewer finishes, read the first line of its file and set that verdict on the board (`VERDICT: PASS` -> `PASS`, `VERDICT: CHANGES` -> `CHANGES`). A missing file or a bad first line: respawn that one reviewer once; if it fails again, mark the job `needs-human` with the reason.
4. **WAIT FOR ALL** chosen reviewers, with no per-reviewer acknowledgement turns. The round is done only when every chosen file exists with a valid verdict. Never resume the builder, or decide anything, before that.
5. Outcome:
   - all chosen PASS: stage `approve`, go to Approve.
   - any CHANGES and n < maxRound: push the reviews onto `past`, go to Resume builder with `JOB/round-n`.
   - any CHANGES and n = maxRound: write `JOB/decision.md` yourself: `ESCALATE`, a blank line, "Review rounds exhausted. Open CHANGES:" and one line per CHANGES review. Decision `ESCALATE`, stage `needs-human`, `note` = that reason.

## Resume builder
Stage `build`. Send to the same builder with SendMessage (never spawn a second one): "Round folder: `<round folder>`. Read every file in it, fix, commit in TREE, append to `JOB/build.md`." Wait for the reply, check the branch has new commits, then run Review round n+1. If the builder is no longer reachable (new session), spawn `builder` again with the same name and prompt plus "continue the work already on the branch; read `build.md` and the round folder".

## Approve
Spawn `approver` with `JOB`, `TREE`, branch, output `JOB/decision.md`. Read the first line.
- `APPROVE`: decision `APPROVE`. If `hold` is true: stage `needs-human`, `note` = "approved, held for manual testing in <TREE>"; keep worktree and branch, do not merge. Otherwise go to Merge.
- `ESCALATE`: decision `ESCALATE`, stage `needs-human`, `note` = the first reasoning line.

## Merge
From `ROOT` on MAIN: `git merge --no-ff factory/<job-id> -m "Merge factory/<job-id>: <title>"`, following the user's commit attribution rules. On any conflict or refusal, `git merge --abort`, stage `needs-human`, `note` = why. On success: stage `merged`, `git worktree remove TREE`, then `git branch -d factory/<job-id>` (safe delete; if it refuses, keep the branch and put why in `note`). Write the board, then record the job on MAIN: `git add factory/board.json JOB` (screenshots are git-ignored) and `git commit -m "Factory: record job <job-id>"`; stage nothing else. Then, only after a successful merge and only if "Branch and push" says to push, push MAIN and nothing else with the command given there. If the push fails, the job stays `merged`, `note` = "merged locally, push failed: <why>"; do not retry with force. Never force anything.

## Finish
End with at most five lines: job id, stage, rounds used, the `note` if any, and the next command the user may want (`/factory approve <id>` for needs-human).

## Rules
- The factory is for code changes only. If the request is for mockups or screenshots of a design, do not start a job: tell the user to make them outside the factory.
- One job at a time per orchestrator. You are the only writer of `board.json`; reviewers, spec-writer and approver write only their own file; the builder writes code in TREE and `build.md`. The only other files you write are `decision.md` when rounds run out, `rework.md`, and the files `init` creates.
- Never read, print, edit or commit env files or credentials. The only allowed action is the plain copy of env files into the worktree in New job step 3.
- The only push allowed is the MAIN push in Merge, after a successful merge. Never push a `factory/*` branch, never use `--force`, `--all`, `--tags` or `-u` with another ref, and never push from any other step or agent.
- Report what happened, including failures and anything agents said they did not verify.
