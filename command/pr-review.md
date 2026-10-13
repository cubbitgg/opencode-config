---
description: Triage a GitHub PR into a numbered Critical/High/Medium/Low findings list (no posting until approved)
agent: build
---

Use the `pr-review` skill to run this workflow end to end.

Arguments: `$ARGUMENTS`

Accepted forms:

```
/pr-review <pr-url>
/pr-review <repo> <pr-id>
```

Examples:

- `/pr-review https://github.com/cubbit/cubbit/pull/4321`
- `/pr-review cubbit 4321`
- `/pr-review devops 88`
- `/pr-review devops-platform 12`

Rules that always apply:

- Follow the `pr-review` skill exactly.
- Do NOT post, comment on, approve, or request changes on the PR in the first
  pass. Produce the numbered findings list and wait for the user.
- NEVER submit a review without explicit, itemized approval of the exact
  comments and the exact command.
- Read-only GitHub only; NEVER run `gh pr checkout` or any git command that
  mutates the working tree, refs, or history.
- Check for real best-practice issues: meaningful logging (enough to understand
  what happened and debug failures, not log-everything), clear and simple code,
  error handling, security, and tests.
