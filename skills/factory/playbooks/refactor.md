# Refactor
## When
Restructuring code with no change to what users or callers see.
## Spec
`## Behaviour` says "unchanged", then lists what must stay the same: public APIs, outputs, user-visible behaviour.
## Builder
- Existing tests stay unchanged, except for moves and renames that `build.md` explains.
## Review
- review-code: run `git -C <worktree> diff --stat <main>...HEAD` on the test files. An unexplained change to a test's assertions is CHANGES with `Rule:` quoting this line.
## Approver
- Escalate if any public API or user-visible behaviour changed.
