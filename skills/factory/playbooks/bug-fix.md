# Bug fix
## When
Something that worked, or should work, behaves wrong.
## Spec
Replace `## Behaviour` with `## Repro`: the steps, the actual result and the expected result.
## Builder
- The first commit adds a test that fails because of the bug, and no fix. Record the commit hash and the failing output of that test in `build.md`.
- The fix comes in a later commit. The test passes at HEAD.
## Review
- review-code: check that the test fails at the test commit and passes at HEAD. Never change the worktree's checkout, because other reviewers run in parallel. Use a temporary worktree under the round folder: `git -C <worktree> worktree add --detach <round folder>/bug-fix-check <test commit>`, run the install command from "Checks" there, run the test, then `git -C <worktree> worktree remove <round folder>/bug-fix-check`. If the remove refuses, say so under `Notes`. Run the test at HEAD in the worktree.
- review-code: a test that does not fail at the test commit, or does not pass at HEAD, is CHANGES with `Rule:` quoting this line.
## Approver
none
