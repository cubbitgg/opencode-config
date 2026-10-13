---
name: pr-review
description: Retrieve a GitHub pull request (from a PR link or a repo name + PR ID) and produce a numbered, severity-categorized findings list (Critical, High, Medium, Low) WITHOUT posting anything. Use when the user runs /pr-review, or asks to review/analyze a pull request, check a PR, or triage PR findings. Posting a review only happens after explicit user approval.
---

# PR Review (triage first, post only on approval)

Analyze a pull request and give the user a numbered list of findings, each
tagged Critical / High / Medium / Low. Do NOT post anything to GitHub. The user
picks which numbered findings should become the actual PR review, and only then
(and only with explicit approval) is anything submitted.

## Absolute rules (never break)

1. NEVER submit, post, comment on, approve, request changes on, or otherwise
   write to a PR without the user's explicit, itemized approval. Read-only
   GitHub access is the default and the whole first pass.
2. NEVER commit, push, merge, rebase, fetch, checkout, or otherwise mutate the
   local git working tree/repo. `gh pr checkout` is FORBIDDEN. Review from the
   diff and from files read via `gh api`, never by switching branches.
3. NEVER fabricate code, line numbers, or findings. Quote the actual diff and
   cite real `file:line` locations. If you are unsure, say so.
4. Do not post a review even if the user approved findings earlier — confirm the
   exact final comment set and the exact command in the same turn before running
   it.
5. Ask when the repo or PR number is ambiguous. Do not guess silently.

## Constants

- Workspace root: `/home/gigiozzz/Dev/workspace`
- GitHub org (default): `cubbit`
- GitHub CLI: `gh` (already authenticated as the active account)

## Step 1 - Parse the arguments

Input arrives as `$ARGUMENTS` and is one of two forms:

1. A PR URL: `https://github.com/<owner>/<repo>/pull/<number>`
   (also accept `owner/repo#<number>`).
2. A repo name + PR ID: `<repo> <number>` (e.g. `cubbit 4321`, `devops 88`,
   `devops-platform 12`).

Extract:

- `owner/repo` (GitHub `owner/name`)
- `number` (PR ID)

If either is missing, ask the user. Do not guess.

## Step 2 - Resolve the repo

Given the `<repo>` token:

1. If a URL or `owner/repo` form was given, use it directly.
2. If `<repo>` contains `/`, use it as `owner/repo`.
3. Otherwise, resolve the workspace folder whose git remote basename equals
   `<repo>`:

   ```
   git -C /home/gigiozzz/Dev/workspace/<folder> remote get-url origin
   ```

   The GitHub repo name is the last path segment (minus `.git`). Match that
   against `<repo>`. This is why the folder may differ from the repo name
   (e.g. folder `devops-master` -> repo `cubbit/devops`, folder `cubbit` ->
   `cubbit/cubbit`).
4. If nothing matches, fall back to `<default-org>/<repo>` and CONFIRM with the
   user before using it.

Confirm the resolved `owner/repo` in your report.

## Step 3 - Fetch the PR (read-only)

Use `gh` with `-R <owner/repo>`:

```
gh pr view <number> -R <owner/repo> --json number,title,author,state,isDraft,url,baseRefName,headRefName,body,additions,deletions,changedFiles,commits,files
gh pr diff <number> -R <owner/repo>
```

Notes:

- The full diff can be large. If truncated, page through it (`gh pr diff` writes
  the whole patch; read it in chunks) or inspect per-file with
  `gh api repos/<owner>/<repo>/pulls/<number>/files`.
- To read a file as it exists on the PR head (for context the diff omits), use:
  `gh api repos/<owner>/<repo>/contents/<path>?ref=<headRefName> --jq .content`
  and base64-decode, or `gh api -H "Accept: application/vnd.github.raw" repos/<owner>/<repo>/contents/<path>?ref=<headRefName>`.
- Read the PR body and commits for intent. Check the linked issue/ticket if the
  body references one, so you judge the change against its goal.

## Step 4 - Review the change

Read every changed hunk. Prioritize correctness and production safety. Apply
this checklist and only report findings that are real and grounded in the diff.

### Correctness & safety

- Bugs, logic errors, off-by-one, nil/null handling, race conditions.
- Edge cases: empty input, missing fields, error paths, retries, timeouts.
- Silent failures: errors swallowed, ignored return values, `except: pass`,
  empty `catch`, unchecked `err`.
- Backwards compatibility: config/API/schema/migration changes.
- Concurrency, resource leaks (unclosed files/connections/goroutines).

### Observability & logging (important)

The bar is NOT "log everything" — it is "a human can understand what happened
and debug a failure from the logs".

- Missing logs on important state transitions or decisions.
- Missing or useless logs on failure/error paths (the most critical gap).
- Logs that cannot be debugged: no context (no IDs/keys/correlation), no
  actionable detail, message that says "error" with nothing else.
- Wrong log level (swallowed error at debug, noisy info in hot loop).
- Secrets/PII/tokens logged. Sensitive data in logs is a security finding.
- Over-logging in hot paths (log spam) is at most a Low/Medium.

### Clarity & simplicity

- Unclear names, misleading names, dead code, duplicated logic, copy-paste.
- Overly clever/complex code where a simple form exists.
- Magic numbers/strings that should be constants/named.
- Unnecessary abstractions, deep nesting, long functions.
- Missing/incorrect comments only where non-obvious logic needs explaining.

### Security

- Hardcoded secrets/keys/tokens, credentials in code or logs.
- Injection (SQL/command/template), unsafe deserialization, path traversal.
- Missing authz/authn checks, overly broad permissions, disabled TLS checks.
- Unpinned/dependency upgrades with known CVEs.

### Tests & maintainability

- Missing tests for new behavior/bug fixes; tests that assert nothing.
- Config/infra changes without validation.
- Docs/changelog updates required by repo convention.

## Step 5 - Assign a severity

Use exactly these four levels:

| Level        | Meaning                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------ |
| **Critical** | Security hole, secret leak, data loss/corruption, production outage, crash in normal path.       |
| **High**     | Likely bug, incorrect behavior, swallowed error hiding failures, missing logs on failure paths.  |
| **Medium**   | Maintainability/clarity problem, missing non-critical logging, edge case, weak test coverage.    |
| **Low**      | Nitpick, naming, style, docs, optional simplification.                                           |

When unsure between two levels, pick the higher one but say why.

## Step 6 - Present the numbered list (do NOT post)

Output a single numbered list. Every item must include:

```
N. [LEVEL] <short title>
   Location: <path>:<line>   (PR head ref: <headRefName>)
   What: <concise explanation grounded in the diff>
   Why it matters: <impact>
   Suggested fix: <concrete, minimal suggestion>
```

Then add a one-line summary count, e.g.
`3 Critical, 2 High, 4 Medium, 1 Low`.

End by asking:

> Which numbers should I include in the PR review? Tell me the numbers (and any
> wording changes). I will NOT post anything until you approve the final set.

Stop and wait. Do nothing else.

## Step 7 - Post the review (ONLY after explicit approval)

Only when the user gives the go-ahead:

1. Show the exact final set of comments: file, line, and the exact comment body
   for each selected number.
2. Show the exact `gh` command(s) you will run.
3. Ask for one final confirmation covering both the content and the command.
4. Only after that confirmation, post.

Prefer an inline review with comments anchored to files/lines:

```
gh api repos/<owner>/<repo>/pulls/<number>/reviews \
  -f event=COMMENT \
  -f body='<overall summary>' \
  -f 'comments[][path]=<path>' \
  -F 'comments[][line]=<line>' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=<comment>'
```

Use `event=COMMENT` unless the user explicitly asks to APPROVE or
REQUEST_CHANGES. If line anchoring is not possible (e.g. unchanged lines),
fall back to a single review `body` with the numbered comments and tell the
user you did so.

After posting, report the review URL. Never post more than the approved set.

## Failure handling

- If `gh` fails (auth, 404, rate limit), stop and show the error; do not retry
  blindly and do not fall back to unauthenticated scraping.
- If the PR does not exist or the number belongs to another repo, stop and ask.
- If the diff cannot be fully retrieved, say exactly which files are missing
  and ask how to proceed.
