# Software factory

A portable [Claude Code](https://claude.com/claude-code) setup that turns an idea into reviewed, merged code. You describe an idea; scouts research the codebase, a spec-writer drafts a spec you iterate on until it's right, then a team of specialised agents builds it on a branch, reviews it, and either merges it or hands it back to you.

Installed once as a Claude Code plugin (agents plus one skill). Everything the factory writes (project config, job state, worktrees) lives in `~/.claude/factory/<project>/`, not in your repo.

## The idea

One AI agent writing and judging its own code is unreliable. The factory splits the work across agents with narrow jobs, and keeps a human in control of anything risky:

```
                             │ /factory <idea>
                             ▼
                   ┌──────────────────┐
                   │  scouts: code +  │
                   │  1 per source    │
                   └─────────┬────────┘
                             ▼ scout-*.md
                   ┌──────────────────┐
                   │   spec-writer    │◄─┐ your changes
                   │       opus       │  │ or answers
                   └─────────┬────────┘  │
                             ▼           │
                   ┌──────────────────┐  │
                   │   spec review    ├──┘
                   │ orchestrator+you │
                   └─────────┬────────┘
                             ▼ you say go
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
              ┌─────────────┐ ┌──────────────────────┐
              │merge to main│ │ needs-human/<job-id> │
              └──────▲──────┘ │ branch + test.md     │
                     │        └──────────┬───────────┘
                     │                   ▼ /factory run, you test
                     └── /factory approve ── you ── /factory rework ─► builder
```

- `review-code` and `review-security` always run. `review-ux` runs only if the diff touches a UX path (listed in `project.md`).
- If CHANGES remain after the second round, the job goes to `needs-human` instead of looping again.
- When a job needs you, the factory creates a local `needs-human/<job-id>` branch and a `test.md` with what to check. `/factory run <job-id>` serves it on a free port; or check the branch out yourself.
- With `--hold`, an APPROVE also stops at `needs-human` so you can test the branch before merging.

| Agent | Job |
|---|---|
| `scout-code` | Researches the code and project docs (current behaviour, patterns to copy, tests, data) and writes one file with file:line references. Sonnet |
| `scout-source` | One per knowledge source in `project.md` (notes folder, Slack, Jira…): searches it for earlier decisions and requirements. Haiku |
| `spec-writer` | Turns the request and the scout findings into a short spec with testable acceptance criteria |
| `builder` | Implements the spec on its own branch and worktree, writes tests, runs the checks |
| `review-code` | Checks correctness, conventions and tests; the only reviewer that runs the test suite |
| `review-ux` | Checks the flow and visual consistency, in a real browser (only when UI files changed) |
| `review-security` | Checks the diff for security problems, including the OWASP Top 10 |
| `approver` | Final gate: `APPROVE` (merge) or `ESCALATE` (needs you) |

Key properties:

- **Isolated.** Each job runs in its own git worktree on a local `factory/<job-id>` branch. Your main checkout is never touched while a job runs.
- **Independent review.** Reviewers run in parallel, only read the diff, and cannot edit code. If any asks for changes, the same builder fixes them, up to two rounds.
- **Humans decide the risky bits.** The approver escalates schema changes, new dependencies, build/infra changes, data deletion, unmet or unchecked acceptance criteria, and anything it is unsure about.
- **Safe git.** Only an approved merge lands on your main branch, and only the main branch is ever pushed. Never `--force`, never a `factory/*` or `needs-human/*` branch. Env files are copied into worktrees but never read or committed.
- **You own the spec.** After the first draft you see its highlights (goal, acceptance criteria, decisions, open questions) and a link to the full `spec.md`. Ask questions or request changes; the spec-writer revises and you see what changed, for as many rounds as it takes. Building starts only when you say go.
- **Everything is files.** Ideas, specs, build notes, reviews and decisions live in `~/.claude/factory/<project>/jobs/<job-id>/`, and `board.json` there tracks job state.

## Setup

Requirements: `git`, a main branch (`main` or similar), and [`playwright-cli`](https://github.com/microsoft/playwright-cli) if your app has a UI (used for browser checks).

1. Install the plugin: `/plugin marketplace add mikkoryynanen/software-factory`, then `/plugin install factory@software-factory`.
2. Open Claude Code in your target repo and run `/factory init` (any `/factory` command runs it first if the repo is not set up). It offers to add `~/.claude/factory` to `permissions.additionalDirectories` in `~/.claude/settings.json` so jobs can read and write their state without prompts, then surveys the repo (docs, package manager, CI, tests, UI code) and drafts `project.md` in `~/.claude/factory/<project>/`: check commands, conventions, which paths trigger the UX reviewer, what to escalate, how to run the app. It asks only what it cannot infer, then offers to add knowledge sources outside the repo (a notes folder, Slack, Jira, Confluence) for the source scouts, including the read-only permissions they need. You confirm the draft and any settings change before anything is written.
3. Dry run with a tiny idea: `/factory --hold <a tiny idea>`.

### Updating

`/plugin marketplace update software-factory`, then `/plugin update factory@software-factory`. Project state in `~/.claude/factory/` is not touched.

### Migrating from the manual copy

If you installed by copying files into `~/.claude/`, remove them before installing the plugin, or you will have duplicate agents and skills:

```
rm -r ~/.claude/skills/factory ~/.claude/agents/factory-*.md
```

Your projects need nothing else: `~/.claude/factory/<project>/` is picked up as is.

`init` also creates `board.json` and `backlog.md` there. In the repo it only adds `.playwright-cli/` to `.git/info/exclude` (local, not committed) when the app has a UI. `<project>` is the repo's absolute path with non-alphanumerics turned into `-`, the same naming Claude Code uses under `~/.claude/projects/`.

## Usage

| Command | What it does |
|---|---|
| `/factory <idea>` | Scout, spec it with you, then run a new job end to end |
| `/factory --hold <idea>` | Same, but stop after approval so you can test the branch before merging |
| `/factory next` | Take the first unchecked item in `backlog.md` and run it (it starts with the spec review too) |
| `/factory continue <job-id>` | Resume a spec review you paused |
| `/factory run <job-id>` | Serve a `needs-human` job's branch locally, open it, and print its test steps |
| `/factory stop [<job-id>]` | Stop that job's review server, or all of them |
| `/factory approve <job-id>` | Merge a job that is waiting on you (`needs-human`), including commits you added to its branch |
| `/factory rework <job-id> <note>` | Send a job back to the builder with your feedback (your commits on `needs-human/<job-id>` go with it) |
| `/factory reject <job-id>` | Drop a `needs-human` job and delete its branches and worktree (asks first) |
| `/factory init` | Set up the factory for this repo |
| `/factory` | Show a one-line-per-job summary of the board |

Typical flow:

1. `/factory add bulk paste to the import dialog`
2. Review the spec highlights, answer its open questions, ask or request changes until it's right, and say go again. Watch it build and review. At the end you get at most five lines: job id, stage, rounds used, a note, and the next command.
3. If the job is `needs-human`, read `test.md`, then either `/factory run <job-id>` to try it in the browser or `git switch needs-human/<job-id>` to review and edit it in your own checkout. Finish with `/factory approve <job-id>`, `/factory rework <job-id> <what to change>` or `/factory reject <job-id>`.

## Configuring for your project

`~/.claude/factory/<project>/project.md` (generated by `init`, edit freely) is the single place that holds project specifics. Every agent reads it. See `skills/factory/project-template.md` for the sections: context docs, branch and push rules, check commands, conventions, reviewer-selection paths, escalation paths, security and UX checklists, and how to run the app in a browser.

## Layout

```
this repo (the plugin)
  .claude-plugin/         plugin.json and marketplace.json
  agents/factory-*.md     the eight agents above
  skills/factory/
    SKILL.md              the orchestrator: the /factory command and the loop
    project-template.md   template for project.md
    browser-check.md      how agents drive the app with playwright-cli

~/.claude/
  factory/<project>/      created by `init`, one folder per repo
    project.md            your project config
    board.json            job state
    backlog.md            queue for `/factory next`
    jobs/<job-id>/        idea.md, scout-*.md, spec.md, spec-feedback.md, build.md, round-N/review-*.md, decision.md, test.md, server.log
    worktrees/<job-id>/   one git worktree per running job
```

## Limits

- Code changes only. It will not produce mockups or design screenshots.
- One job at a time per orchestrator session.
- Quality depends on `project.md`: if your check commands and conventions are vague, so are the reviews.

## Credits

Based on the software factory build prompt from [ZazenCodes, season 3](https://github.com/zazencodes/zazencodes-season-3/blob/main/src/software-factory-claude-code/BUILD_PROMPT.md).
