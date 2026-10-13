# start-jira-worktree reference

## Paths

| Thing          | Path                                                             |
| -------------- | ---------------------------------------------------------------- |
| Workspace root | `/home/gigiozzz/Dev/workspace`                                   |
| Source repo    | `/home/gigiozzz/Dev/workspace/<repo>`                            |
| Worktree       | `/home/gigiozzz/Dev/workspace/worktree/<repo>/<JIRACODE>`        |

Existing examples that already follow this layout:

- `worktree/cubbit/COM-1623` -> `feat/COM-1623-kubectl-plugin-add-support-to-profile-ingress-controller-via-profile-command`
- `worktree/cubbit/COM-1722` -> `fix/COM-1722-operator-inventory-change-default-k-8-s-svc-routing-to-cluster`
- `worktree/cubbit/OPS-2836` -> `chore/OPS-2836-upgrade-golang-and-k8s-libs-and-tests`

## Branch type mapping

| Jira issue type               | Branch type |
| ----------------------------- | ----------- |
| Story, Feature                | `feat`      |
| Bug, Defect                   | `fix`       |
| Task, Sub-task, Chore, Spike  | `chore`     |
| unknown                       | ask user    |

Always show the guess and get confirmation before using it.

## Title normalization

Algorithm: lowercase -> replace every run of non `[a-z0-9]` with `-` -> collapse
`-` -> trim `-`. Common short forms are fine (e.g. `k8s` stays `k8s`).

Examples:

| Jira summary                              | normalized                     | branch (type=feat)                                             |
| ----------------------------------------- | ------------------------------ | ------------------------------------------------------------- |
| `Kubectl plugin: add support to profile ingress-controller via "profile" command` | `kubectl-plugin-add-support-to-profile-ingress-controller-via-profile-command` | `feat/COM-1623-kubectl-plugin-add-support-to-profile-ingress-controller-via-profile-command` |
| `Operator inventory: change default K8s SVC routing to cluster` | `operator-inventory-change-default-k8s-svc-routing-to-cluster` | `fix/COM-1722-...` |
| `Upgrade Golang & K8s libs (and tests!)`  | `upgrade-golang-k8s-libs-and-tests` | `chore/OPS-2836-upgrade-golang-k8s-libs-and-tests`            |
| `Bump v2.0.0`                             | `bump-v2-0-0`                  | `chore/OPS-1234-bump-v2-0-0`                                  |

## Jira MCP tool cheatsheet

The official Atlassian remote MCP server is configured as `jira`
(`https://mcp.atlassian.com/v1/mcp`, OAuth). Exact tool names depend on the
server; the likely names are:

- `getAccessibleAtlassianResources` - get the `cloudId` needed by other calls.
- `getJiraIssue` - fetch an issue by key (fields: summary, issuetype, status,
  duedate, timetracking, description, and the story-points custom field).
- `editJiraIssue` - update fields such as `duedate`, `timetracking`,
  `description`, story points.
- `getTransitionsForJiraIssue` - list transitions available to the current user.
- `transitionJiraIssue` - perform a transition by id.
- `searchJiraIssuesUsingJql` - fallback search.
- `getVisibleJiraProjects`, `lookupJiraAccountId` - supporting lookups.

At runtime, discover the actual tools available (the tool list may differ). If a
name is missing, use the closest available Jira tool rather than skipping the
step. If no suitable tool exists, tell the user.

### Typical call order

1. `getAccessibleAtlassianResources` -> `cloudId`
2. `getJiraIssue({ cloudId, issueIdOrKey: "<JIRACODE>" })`
3. ... confirm type with user, create worktree ...
4. `editJiraIssue(...)` for each confirmed missing field
5. `getTransitionsForJiraIssue({ cloudId, issueIdOrKey })`
6. `transitionJiraIssue({ cloudId, issueIdOrKey, transitionId })`

### Story points

Story points usually live in a custom field (commonly `customfield_10016`,
sometimes returned as `story_points`). Confirm the field name from the issue
payload before writing. If you cannot identify it, ask the user.

## Git commands allowed

Only these:

```
git -C <repo> rev-parse --is-inside-work-tree        # validate
git -C <repo> rev-parse --abbrev-ref origin/HEAD     # default branch (local only)
git -C <repo> branch --list <branch>                 # collision check
git -C <repo> worktree list                          # read
git -C <repo> worktree add -b <branch> <path>        # the ONLY mutation
```

Forbidden: `commit`, `push`, `merge`, `rebase`, `fetch`, `pull`, `reset`,
`checkout` of refs, `branch -D`, and anything else that mutates git state.

## Example run

User: `/start cubbit #COM-1564`

1. repo=`cubbit`, jiracode=`COM-1564`.
2. Validate `/home/gigiozzz/Dev/workspace/cubbit` is a git repo.
3. Fetch COM-1564: summary `Add apex domain validation`, type `Story`, status
   `To Do`, no due date, no estimate, no story points, empty description.
4. Guess `feat`, ask user -> user confirms `feat`.
5. Normalize -> `add-apex-domain-validation`.
6. Branch `feat/COM-1564-add-apex-domain-validation`; no collision.
7. `git -C /home/gigiozzz/Dev/workspace/cubbit worktree add -b feat/COM-1564-add-apex-domain-validation /home/gigiozzz/Dev/workspace/worktree/cubbit/COM-1564`
8. Missing fields: ask for due date, estimate, story points, description. User
   provides some, skips others, confirms before each write.
9. Transition COM-1564 to In Progress.
10. Report branch + path; remind user to commit/push themselves.
