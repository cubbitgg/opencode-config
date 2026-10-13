---
name: start-jira-worktree
description: Create a git worktree for a repo under /home/gigiozzz/Dev/workspace from a Jira issue code, using branch <type>/<JIRACODE>-<normalized-title>, then move the Jira issue to In Progress. Use when the user runs /start <repo> #<JIRACODE> or asks to "start" a Jira task as a worktree.
---

# Start Jira Worktree

Turn a Jira issue into a ready-to-work git worktree. This skill is driven by the
`/start` command, but it can also be triggered directly when the user says
something like "start COM-1564 on cubbit as a worktree".

## Absolute rules (never break)

1. The ONLY mutating git command allowed is `git worktree add`. Read-only git is
   fine (`git worktree list`, `git status`, `git rev-parse`, `git branch`).
2. NEVER run `git commit`, `git push`, `git merge`, `git rebase`, `git fetch`,
   `git pull`, `git reset`, or anything else that mutates refs or history. The
   user commits and pushes by themselves.
3. Ask before guessing. If a value is unknown or ambiguous, ask the user.
4. Never fabricate Jira data. Only use what the Jira MCP returns.

## Constants

- Workspace root: `/home/gigiozzz/Dev/workspace`
- Repo source: `/home/gigiozzz/Dev/workspace/<repo>`
- Worktree root: `/home/gigiozzz/Dev/workspace/worktree/<repo>/<JIRACODE>`
- Branch format: `<type>/<JIRACODE>-<normalized-title>`
- Types: `feat`, `chore`, `fix`

## Step 1 - Parse arguments

Input arrives as `$ARGUMENTS` of the form `<repo> #<JIRACODE>`.

- `repo` = first token (a folder name directly under the workspace root).
- `jiracode` = the token starting with `#`, stripped of `#`, uppercased.
  Also accept it without `#`. Normalize `#com-1564` -> `COM-1564`.

If either is missing, ask the user for it. Do not guess the repo.

## Step 2 - Validate the repo

- Confirm `/home/gigiozzz/Dev/workspace/<repo>` exists and is a git repository
  (for example `git -C <path> rev-parse --is-inside-work-tree`).
- If it is not a git repo, stop and tell the user.

## Step 3 - Fetch the Jira issue

Use the Atlassian/Jira MCP tools. See `reference.md` for the tool cheatsheet.

1. Get the `cloudId` via the accessible-resources tool if required.
2. Fetch the issue and read: `summary`, `issuetype`, `status`, `duedate`,
   time tracking / original estimate, story points, and `description`.

## Step 4 - Guess the branch type, then ask

Map the issue type to a branch type:

| Jira issue type            | type    |
| -------------------------- | ------- |
| Story, Feature             | `feat`  |
| Bug, Defect                | `fix`   |
| Task, Sub-task, Chore, Spike | `chore` |

Show the guess and ASK the user to confirm before using it:

> "Jira type is `Story`, so I'll use branch type `feat`. OK, or do you want
> `chore` / `fix`?"

Wait for the answer. If the issue type is unknown, present all three options.

## Step 5 - Normalize the title

Normalize the Jira summary:

1. Lowercase.
2. Replace any run of characters that are not `a-z` or `0-9` with a single `-`.
3. Collapse repeated `-` and trim leading/trailing `-`.

Result: `<normalized-title>`. Examples live in `reference.md`.

`<branch>` = `<type>/<JIRACODE>-<normalized-title>`.

Trim the branch to a sane length (about 80 chars) by cutting on a `-` boundary.

## Step 6 - Check collisions before creating

- `git -C /home/gigiozzz/Dev/workspace/<repo> worktree list` - is the target path
  already a worktree?
- `git -C /home/gigiozzz/Dev/workspace/<repo> branch --list <branch>` - does the
  branch already exist?

If either exists, STOP and ask the user how to proceed (pick another name, reuse
the existing worktree, or abort). Never force or overwrite.

## Step 7 - Create the worktree (the only mutating git command)

Base the new branch on the repo's local default branch when determinable (for
example `git -C <path> rev-parse --abbrev-ref origin/HEAD`), otherwise on the
repo's current HEAD. If unclear, ask the user which base to use. Do NOT fetch.

Run exactly:

```
git -C /home/gigiozzz/Dev/workspace/<repo> worktree add -b <branch> /home/gigiozzz/Dev/workspace/worktree/<repo>/<JIRACODE>
```

Then confirm the path and branch with `git worktree list`. Do not run any other
mutating git command.

## Step 8 - Jira hygiene: due date, estimate, story points, description

Check whether the issue is missing any of: due date, original estimate / time
tracking, story points, description.

For EACH missing field:

1. Ask the user for the value (or whether to skip it).
2. Show exactly what you will write and ask for confirmation.
3. Only after confirmation, update the issue via the Jira MCP edit tool.

Never write a value the user did not explicitly confirm. It is fine to leave a
field empty if the user says to skip.

## Step 9 - Move the issue to In Progress

1. Get the available transitions for the issue.
2. Find the transition named "In Progress" (case-insensitive). Prefer an exact
   match.
3. If found, perform it. If not found or ambiguous, ask the user which
   transition to use. Do not guess into an arbitrary transition.

## Step 10 - Report

Give a short summary:

- Branch: `<branch>`
- Worktree: `/home/gigiozzz/Dev/workspace/worktree/<repo>/<JIRACODE>`
- Jira: `<JIRACODE>` moved to In Progress (or why not)
- Any fields skipped

Remind the user that git commits/pushes are on them.

## Failure handling

- If any git or MCP call fails, stop, show the error, and ask how to proceed.
- Never retry a mutating git command automatically.
