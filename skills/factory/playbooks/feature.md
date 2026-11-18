# Feature
## When
A new capability, or a change to what users can do.
## Spec
The standard spec. Write `## Design` before the acceptance criteria.
## Builder
- Start from `## Design`: build the data shape before the logic.
- One test per acceptance criterion that code can check.
- Small ordered commits. Each commit passes the commands in "Checks".
## Review
- review-code: the diff stays within "Affected modules", and each acceptance criterion that code can check has a test.
## Approver
none
