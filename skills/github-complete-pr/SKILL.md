---
name: github-complete-pr
description: "Use when a GitHub pull request has just been merged or closed - switches back to the repository default branch, syncs it, and deletes the now-obsolete feature branch both locally and on the remote if it still exists"
---

# GitHub Complete PR

## Overview

Once a pull request is merged or closed, the feature branch is no longer needed. Leaving it around clutters the local repo and the remote, creates stale references, and confuses future `git branch` / PR listings. This skill defines the cleanup steps to run immediately after PR completion.

## When to Use

- A pull request has just been merged (squash, rebase, or merge commit)
- A pull request has been closed without merging and the branch will not be reused
- The user says things like "PR is merged", "done with that PR", "close it out", "clean up the branch"

## Do NOT Use When

- The PR is still open or in draft
- The branch is shared with other collaborators who still have in-flight work on it
- The branch is a long-lived branch (e.g. `develop`, `release/*`, `main`) — never delete these

## Rules

### 1. Confirm the PR is actually merged or closed

Before deleting anything, verify the PR state via `gh`:

```bash
gh pr view <number-or-branch> --json state,mergedAt,headRefName,baseRefName
```

Only proceed when `state` is `MERGED` or `CLOSED`. If the PR is still `OPEN`, stop and tell the user.

### 2. Identify the repository default branch

Do not assume `main`. Query it:

```bash
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
```

Use whatever it returns (`main`, `master`, `trunk`, etc.) as the checkout target.

### 3. Switch back to the default branch and sync

```bash
git checkout <default-branch>
git pull --ff-only origin <default-branch>
```

Use `--ff-only` to avoid accidental merge commits if the local default branch has drifted.

### 4. Delete the local feature branch

```bash
git branch -d <feature-branch>
```

Use `-d` (safe delete) first. If Git refuses because the branch is not fully merged into the default branch (common when the PR was squash-merged), confirm the PR was actually merged via `gh pr view`, then force the delete:

```bash
git branch -D <feature-branch>
```

Never use `-D` without first confirming PR merge/close state.

### 5. Delete the remote branch if it still exists

Many PR workflows auto-delete the remote branch on merge, but not all. Check and clean up:

```bash
git fetch --prune origin
git ls-remote --exit-code --heads origin <feature-branch> && \
  git push origin --delete <feature-branch> || \
  echo "Remote branch already deleted"
```

`--prune` removes stale remote-tracking refs. The `ls-remote` check avoids a noisy error when the remote branch is already gone.

### 6. Confirm cleanup

Finish by showing the user the state so they can verify:

```bash
git branch            # local branches
git branch -r         # remote-tracking branches
```

The feature branch must be absent from both.

## Full Recipe

```bash
# 1. Verify PR state
gh pr view <branch-or-number> --json state,headRefName

# 2. Find default branch
DEFAULT=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)

# 3. Switch + sync
git checkout "$DEFAULT"
git pull --ff-only origin "$DEFAULT"

# 4. Delete local branch (fallback to -D after confirming merge)
git branch -d <feature-branch> || git branch -D <feature-branch>

# 5. Prune + delete remote branch if still present
git fetch --prune origin
git ls-remote --exit-code --heads origin <feature-branch> \
  && git push origin --delete <feature-branch> \
  || echo "Remote branch already deleted"

# 6. Verify
git branch
git branch -r
```

## Red Flags — STOP

- PR state is `OPEN` → do not delete anything
- Current branch has uncommitted changes → stash or commit before switching
- Feature branch name matches the default branch or a protected pattern (`main`, `master`, `develop`, `release/*`) → never delete
- Force-delete (`-D`) is needed but PR merge state is unverified → verify first
