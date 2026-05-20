---
name: pr-risk-review
description: Review a pull request for user impact, operational risk, missing tests, rollback readiness, and release notes.
---

# PR Risk Review

Use when asked to review a pull request before merge.

Check the diff, tests, deployment path, and touched ownership areas. Focus on behavior changes, data migrations, permission changes, external calls, background jobs, and observability gaps.

Return:

1. Blocking risks with file references.
2. Non-blocking concerns.
3. Missing tests or validation.
4. Rollback note.
5. A merge recommendation: `merge`, `merge after fixes`, or `do not merge yet`.

Keep the review specific to the PR. Do not request broad refactors.

