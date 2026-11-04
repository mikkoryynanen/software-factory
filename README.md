# Software factory (portable copy)

1. Copy the `.claude/` folder from here into the root of the target repo (merge with any existing `.claude/`).
2. In that repo, open Claude Code and run `/factory init`. It drafts `factory/project.md` for the repo and asks you to confirm it.
3. Dry run: `/factory --hold <a tiny feature>`.

Needs: git, a `main`-style branch, and `playwright-cli` if the app has a UI.
