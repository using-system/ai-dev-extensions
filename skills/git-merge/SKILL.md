---
name: git-merge
description: "Use when performing any git merge operation (direct merge, PR merge, or MR merge) - requires explicit user confirmation before executing the merge command"
---

# Git Merge with User Confirmation

## Overview

Every git merge — whether a direct `git merge`, a GitHub PR merge via `gh pr merge`, or a GitLab MR merge — MUST be preceded by explicit user confirmation. Merges are hard to reverse cleanly and can affect shared branches. This skill ensures the user always makes the final call.

## When to Use

- Running `git merge` directly (any branch into any branch)
- Merging a pull request via `gh pr merge` or the GitHub API
- Merging a merge request via `glab mr merge` or the GitLab API
- Any workflow step that results in a merge commit, fast-forward merge, or squash merge

## Do NOT Use When

- The user is only *viewing* merge status or conflicts (`git log --merges`, `gh pr status`)
- The user explicitly asks to skip confirmation ("just merge it", "merge without asking")
- Rebasing — this is not a merge (use caution separately, but this skill does not apply)

## Rules

### 1. Never merge without explicit user confirmation

Before executing any merge command, present a summary and ask the user to confirm.

**Required summary contents:**

| Field | Description |
|-------|-------------|
| **Source** | The branch or ref being merged in |
| **Target** | The branch receiving the merge |
| **Strategy** | Merge commit, squash, or fast-forward |
| **Commits** | Number of commits being merged |
| **Conflicts** | Whether conflicts are expected (run a dry-run first) |

```bash
# Dry-run to detect conflicts before asking
git merge --no-commit --no-ff <source-branch> 2>&1
git merge --abort 2>/dev/null
```

### 2. Ask clearly and wait

Present the summary, then ask a direct yes/no question. Do not proceed until the user explicitly confirms.

```
Merge summary:
  Source: feat/oauth2-login
  Target: main
  Strategy: merge commit
  Commits: 4
  Conflicts: none detected

Proceed with merge? (yes/no)
```

### 3. Respect "no"

If the user declines, stop. Do not retry, rephrase, or suggest alternatives unless the user asks for them.

### 4. Check the target branch before merging

Confirm you are on the correct target branch before merging:

```bash
git branch --show-current
```

If the current branch is not the intended target, switch to it first — but confirm with the user if the target is a protected branch (`main`, `master`, `develop`).

### 5. PR/MR merges follow the same rule

When merging via `gh pr merge` or `glab mr merge`, the same confirmation is required:

```
PR merge summary:
  PR: #42 — feat: add OAuth2 login flow
  Target: main
  Strategy: squash
  CI status: all checks passed
  Approved: 2/2 reviewers

Proceed with merge? (yes/no)
```

### 6. Post-merge verification

After a successful merge, report the result:

```bash
# Show the merge commit
git log --oneline -1

# Confirm current branch state
git branch --show-current
```

## Examples

```bash
# BAD — merging without asking
git merge feat/new-api

# BAD — PR merge without confirmation
gh pr merge 42 --squash

# GOOD — dry-run, summarize, confirm, then merge
git merge --no-commit --no-ff feat/new-api
git merge --abort
# ... present summary and ask user ...
# user confirms
git merge feat/new-api

# GOOD — PR merge with confirmation
# ... present PR summary and ask user ...
# user confirms
gh pr merge 42 --squash
```

## Red Flags — STOP

- About to run `git merge` or `gh pr merge` without having asked the user → stop and ask first
- User said "no" and you are about to merge anyway → stop immediately
- Merging into a protected branch (`main`, `master`) without confirming the target → verify the branch first
- Merge has conflicts and you are about to force it → present the conflicts to the user and let them decide
