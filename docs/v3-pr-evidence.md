# v3 idea: PR with evidence

## Problem
After a job is approved, the human has to dig through `~/.claude/factory/<SLUG>/` to see what was built and how it was tested. Without `--hold` the work is merged without a human look at all.

## Idea
- Hold before merge by default (`--auto` to opt out).
- `/factory pr <job-id>`: assemble `pr.md` from `idea.md`, `build.md`, the reviews and `decision.md` (what was added, acceptance criteria table, how each was checked, "Not verified" list), plus screenshots and a demo video. Print the `gh pr create` command; the user pushes and opens the PR (the factory never pushes `factory/*`).
- Builder records the demo during its existing acceptance-criteria browser check: `playwright-cli video-start`, `video-chapter <criterion>`, `video-stop`; `ffmpeg` converts to mp4/gif.
- Reuse existing `screens/<role>/` screenshots instead of taking new ones.

## Token cost (measured on job 006, backend-only)
- Whole job: ~44k output, ~2.6M cache-read, ~204k cache-write tokens. Builder and orchestrator dominate.
- PR description: about +10-15k input, +1.5-2.5k output. Under 1% of total.
- Screenshots: ~1.4k tokens per Read at 1280x800, and they stay in context. Reuse, don't add.
- Video: ~0 model tokens as long as no agent reads it or extracts frames.
- Avoid: an extra agent session for PR evidence, `snapshot` in the demo run, viewing the video.
- Not measured: a UI job.

## Native (Kotlin/KMP) apps
`playwright-cli` is browser-only, so it cannot record native apps.
- Android: `adb shell screenrecord` (3 min cap), needs emulator/device.
- iOS: `xcrun simctl io booted recordVideo`.
- The flow must be driven by something: a UI test per UI-facing criterion (Compose UI test, Espresso, XCUITest) run while the recorder runs. Maestro would be a new dependency, so ask first.
- `project.md` needs a "Native recording" section: recorder and test command. No emulator means "video not recorded" in the PR.

## Open questions
- Where media lives: committed under `docs/pr/<job-id>/` (repo bloat), gist/release asset, or attached by hand.
- Stage for a job whose PR is open (`pr-open`?) and worktree cleanup after the PR merges.
