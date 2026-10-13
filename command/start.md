---
description: Start a Jira task as a git worktree (branch + worktree + Jira In Progress)
agent: build
---

Use the `start-jira-worktree` skill to run this workflow end to end.

Arguments: `$ARGUMENTS`

Expected form:

```
/start <repo> #<JIRACODE>
```

Example: `/start cubbit #COM-1564`

Rules that always apply:

- Follow the `start-jira-worktree` skill exactly, including its confirmation prompts.
- The only git command you may run is `git worktree add` (plus read-only git such as
  `git worktree list`, `git status`, `git rev-parse`). NEVER commit, push, merge,
  rebase, fetch, or otherwise mutate git state. The user handles git themselves.
