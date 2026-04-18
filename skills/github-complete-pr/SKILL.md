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

Use `-d` (safe delete) first. Git will refuse when the branch is not fully merged into the default branch — this is common and expected when the PR was squash-merged or rebase-merged (the tip commit of the feature branch is not an ancestor of `main`).

Force-delete with `-D` is acceptable **only** when one of the following is true, confirmed via `gh pr view`:

- `state` is `MERGED` — the changes exist on the default branch under a new commit hash; the original commits are redundant.
- `state` is `CLOSED` **and** the branch was intentionally abandoned (the user confirms the work will not be reused).

```bash
git branch -D <feature-branch>
```

**Data-loss warning.** For a `CLOSED` (unmerged) PR, `-D` permanently discards any commits on that branch that live nowhere else — there is no remote copy to recover from once the remote branch is also deleted in step 5. Confirm with the user before force-deleting a closed-unmerged branch.

Never use `-D` without first confirming PR state via `gh pr view`.

### 5. Delete the remote branch if it still exists

Many PR workflows auto-delete the remote branch on merge, but not all. Check and clean up:

```bash
git fetch --prune origin
if git ls-remote --exit-code --heads origin <feature-branch> >/dev/null 2>&1; then
  git push origin --delete <feature-branch>
else
  echo "Remote branch already deleted"
fi
```

`--prune` removes stale remote-tracking refs. The `ls-remote` check avoids a noisy error when the remote branch is already gone. The explicit `if/else` (instead of `A && B || C`) ensures a real `git push --delete` failure surfaces as an error rather than being swallowed by the "already deleted" message.

### 6. Confirm cleanup

Finish by showing the user the state so they can verify:

```bash
git branch            # local branches
git branch -r         # remote-tracking branches
```

The feature branch must be absent from both.

## Full Recipe

The recipe below enforces Rule §1 (abort when `state` is `OPEN`) and Rule §4 (only force-delete after state validation), and derives `FEATURE_BRANCH` from the PR itself so a mistyped argument cannot cause the wrong branch to be deleted.

```bash
PR=<branch-or-number>

# 1. Verify PR state and capture the exact head branch from the PR
PR_STATE=$(gh pr view "$PR" --json state -q .state)
FEATURE_BRANCH=$(gh pr view "$PR" --json headRefName -q .headRefName)

if [ -z "$PR_STATE" ] || [ -z "$FEATURE_BRANCH" ]; then
  echo "Could not determine PR state or head branch; aborting."
  exit 1
fi

if [ "$PR_STATE" = "OPEN" ]; then
  echo "PR is still OPEN; refusing to delete anything."
  exit 1
fi

# 2. Find default branch and refuse protected branches
DEFAULT=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)

if [ "$FEATURE_BRANCH" = "$DEFAULT" ] \
    || [ "$FEATURE_BRANCH" = "main" ] \
    || [ "$FEATURE_BRANCH" = "master" ] \
    || [ "$FEATURE_BRANCH" = "develop" ] \
    || [[ "$FEATURE_BRANCH" == release/* ]]; then
  echo "Refusing to delete protected or default branch: $FEATURE_BRANCH"
  exit 1
fi

# 3. Switch + sync
git checkout "$DEFAULT"
git pull --ff-only origin "$DEFAULT"

# 4. Delete local branch — safe delete first; auto-force-delete only for MERGED PRs.
#    For CLOSED (unmerged) PRs, abort and require the user to explicitly confirm data loss.
if ! git branch -d "$FEATURE_BRANCH"; then
  if [ "$PR_STATE" = "MERGED" ]; then
    echo "Safe delete failed (squash/rebase merge rewrote history). Force-deleting merged branch: $FEATURE_BRANCH"
    git branch -D "$FEATURE_BRANCH"
  else
    echo "Safe delete failed because the branch is not fully merged."
    echo "PR state is $PR_STATE (not MERGED). Refusing to force-delete without explicit confirmation that the work is abandoned."
    echo "If you have confirmed the branch can be discarded, rerun manually:"
    echo "  git branch -D \"$FEATURE_BRANCH\""
    exit 1
  fi
fi

# 5. Prune + delete remote branch if still present
git fetch --prune origin
if git ls-remote --exit-code --heads origin "$FEATURE_BRANCH" >/dev/null 2>&1; then
  git push origin --delete "$FEATURE_BRANCH"
else
  echo "Remote branch already deleted"
fi

# 6. Verify
git branch
git branch -r
```

## Red Flags — STOP

- PR state is `OPEN` → do not delete anything
- Current branch has uncommitted changes → stash or commit before switching
- Feature branch name matches the default branch or a protected pattern (`main`, `master`, `develop`, `release/*`) → never delete
- Force-delete (`-D`) is needed but PR merge state is unverified → verify first
