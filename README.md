# Software factory

A portable [Claude Code](https://claude.com/claude-code) setup that turns a one-line feature request into reviewed, merged code. You describe a feature; a team of specialised agents writes the spec, builds it on a branch, reviews it, and either merges it or hands it back to you.

Nothing to install: it is just a `.claude/` folder (agents plus one skill) that you copy into your own repo.

## The idea

One AI agent writing and judging its own code is unreliable. The factory splits the work across agents with narrow jobs, and keeps a human in control of anything risky:

```
                   ┌──────────────────┐
                   │     feature      │
                   └─────────┬────────┘
                             ▼
                   ┌──────────────────┐
                   │   spec-writer    │
                   │       opus       │
                   └─────────┬────────┘
                             ▼
                   ┌──────────────────┐
                   │     builder      │◄───────────────────┐
                   │       opus       │                    │
                   └─────────┬────────┘                    │
              ┌──────────────┴──────────────┐              │
              ▼              ▼              ▼              │
        ┌───────────┐  ┌───────────┐  ┌───────────┐        │
        │ security  │  │     ux    │  │    code   │        │
        │    opus   │  │   sonnet  │  │   sonnet  │        │
        │(if needed)│  │(if needed)│  │ (always)  │        │
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘        │
              └──────────────┬──────────────┘              │
                             ▼                             │
                   ┌──────────────────┐   any CHANGES:     │
                   │   WAIT FOR ALL   ├────────────────────┘
                   │  chosen reviews  │   resume builder
                   └─────────┬────────┘   (max 2 rounds)
                             ▼ all PASS
                   ┌──────────────────┐
                   │     approver     │
                   │       opus       │
                   └─┬───────────────┬┘
             APPROVE │               │ ESCALATE
                     ▼               ▼
              ┌─────────────┐ ┌─────────────┐
              │merge to main│ │ needs-human │ ◄── /factory approve
              └─────────────┘ └─────────────┘
```

- `review-code` always runs. `review-ux` runs only if the diff touches a UX path, `review-security` only if it touches a security path (both lists are in `factory/project.md`).
- If CHANGES remain after the second round, the job goes to `needs-human` instead of looping again.
- With `--hold`, an APPROVE also stops at `needs-human` so you can test the branch before merging.

| Agent | Job |
|---|---|
| `spec-writer` | Turns the request into a short spec with testable acceptance criteria |
| `builder` | Implements the spec on its own branch and worktree, writes tests, runs the checks |
| `review-code` | Checks correctness, conventions and tests; the only reviewer that runs the test suite |
| `review-ux` | Checks the flow and visual consistency, in a real browser (only when UI files changed) |
| `review-security` | Checks the diff for security problems (only when sensitive paths changed) |
| `approver` | Final gate: `APPROVE` (merge) or `ESCALATE` (needs you) |

Key properties:

- **Isolated.** Each job runs in its own git worktree on a local `factory/<job-id>` branch. Your main checkout is never touched while a job runs.
- **Independent review.** Reviewers run in parallel, only read the diff, and cannot edit code. If any asks for changes, the same builder fixes them, up to two rounds.
- **Humans decide the risky bits.** The approver escalates schema changes, new dependencies, build/infra changes, data deletion, unmet or unchecked acceptance criteria, and anything it is unsure about.
- **Safe git.** Only an approved merge lands on your main branch, and only the main branch is ever pushed. Never `--force`, never a `factory/*` branch. Env files are copied into worktrees but never read or committed.
- **Everything is files.** Specs, build notes, reviews and decisions live in `factory/jobs/<job-id>/`, and `factory/board.json` tracks job state. You can read the whole history in git.

## Setup

Requirements: `git`, a main branch (`main` or similar), and [`playwright-cli`](https://github.com/microsoft/playwright-cli) if your app has a UI (used for browser checks).

1. Copy the `.claude/` folder from this repo into the root of your target repo (merge with any existing `.claude/`).
2. Open Claude Code in that repo and run `/factory init`. It surveys the repo (docs, package manager, CI, tests, UI code) and drafts `factory/project.md`: check commands, conventions, which paths trigger which reviewer, what to escalate, how to run the app. It asks only what it cannot infer, and you confirm the draft before anything is written.
3. Dry run with a tiny feature: `/factory --hold <a tiny feature>`.

`init` also creates `factory/board.json` and `factory/backlog.md`, adds `.claude/worktrees/` to `.gitignore`, and adds a short "Software factory" section to your `CLAUDE.md`. It does not commit anything.

## Usage

| Command | What it does |
|---|---|
| `/factory <feature>` | Run a new job end to end |
| `/factory --hold <feature>` | Same, but stop after approval so you can test the branch before merging |
| `/factory next` | Take the first unchecked item in `factory/backlog.md` and run it |
| `/factory approve <job-id>` | Merge a job that is waiting on you (`needs-human`) |
| `/factory rework <job-id> <note>` | Send a job back to the builder with your feedback |
| `/factory init` | Set up the factory in this repo |
| `/factory` | Show a one-line-per-job summary of the board |

Typical flow:

1. `/factory add bulk paste to the import dialog`
2. Watch it spec, build and review. At the end you get at most five lines: job id, stage, rounds used, a note, and the next command.
3. If the job is `needs-human`, inspect the worktree at `.claude/worktrees/<job-id>`, then `/factory approve <job-id>` or `/factory rework <job-id> <what to change>`.

## Configuring for your project

`factory/project.md` (generated by `init`, edit freely) is the single place that holds project specifics. Every agent reads it. See `.claude/skills/factory/project-template.md` for the sections: context docs, branch and push rules, check commands, conventions, reviewer-selection paths, escalation paths, security and UX checklists, and how to run the app in a browser.

## Layout

```
.claude/
  agents/                 the six agents above
  skills/factory/
    SKILL.md              the orchestrator: the /factory command and the loop
    project-template.md   template for factory/project.md
    browser-check.md      how agents drive the app with playwright-cli
factory/                  created in your repo by `init`
  project.md              your project config
  board.json              job state
  backlog.md              queue for `/factory next`
  jobs/<job-id>/          spec.md, build.md, round-N/review-*.md, decision.md
```

## Limits

- Code changes only. It will not produce mockups or design screenshots.
- One job at a time per orchestrator session.
- Quality depends on `factory/project.md`: if your check commands and conventions are vague, so are the reviews.

## Credits

Based on the software factory build prompt from [ZazenCodes, season 3](https://github.com/zazencodes/zazencodes-season-3/blob/main/src/software-factory-claude-code/BUILD_PROMPT.md).
