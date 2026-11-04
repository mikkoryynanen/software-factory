# Browser check (factory agents)

Use `playwright-cli` (token-efficient; no MCP schemas). If `playwright-cli` is not on PATH, say so in your file and list the browser checks as not verified. Never claim a check you did not run, and do not install anything yourself.

How to serve this app, sign in, and what to watch for are in the "App in the browser" section of `factory/project.md`. Viewports are in its "UX checklist". The orchestrator copies the main checkout's env files into each worktree; never create, edit or read them. If the app fails for missing env values, say so and list the browser checks as not verified.

The app is real DOM: use `snapshot` refs, not coordinates.

## Ports and sessions
Ports: builder 8101, ux 8102, code 8104. Pick another free port if taken. Use a named session `-s=<job-id>-<role>` so each agent starts signed out, and a throwaway account per agent (`<job-id>-<role>@example.test`) so parallel agents never share data.

## Drive it
```bash
playwright-cli -s=<job-id>-<role> open --browser=chromium http://localhost:<port>/
playwright-cli -s=<job-id>-<role> resize <w> <h>
playwright-cli -s=<job-id>-<role> snapshot
playwright-cli -s=<job-id>-<role> click <ref>
playwright-cli -s=<job-id>-<role> fill <ref> "text"
playwright-cli -s=<job-id>-<role> press Enter
playwright-cli -s=<job-id>-<role> screenshot --filename=<out>.png
playwright-cli -s=<job-id>-<role> close
```
Look at the screenshots (Read the PNG), not just the command output.

## Finish
`playwright-cli -s=<name> close`, then stop your preview server. Save screenshots under `<round folder or job folder>/screens/<role>/`. List in your file which screens you saw and which you did not.
