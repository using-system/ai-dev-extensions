---
name: git-push
description: "Use when pushing changes to a remote repository - runs a local code review before pushing, fixes any issues found, and repeats until the review passes clean"
---

# Git Push with Pre-Push Review

## Overview

Before pushing changes to a remote repository, a local code review MUST be performed first. This catches problems earlier — before they reach the remote and before CI runs. The skill runs the active AI coding assistant's review command (e.g., `/review` in Copilot CLI, Claude Code, or equivalent), fixes any issues found, commits those fixes, and repeats the review cycle until the code is clean. Only then does the push proceed.

## When to Use

- Running `git push` to any remote branch
- Pushing after committing fixes, features, or refactors
- Any workflow step that sends local commits to a remote

## Do NOT Use When

- The user explicitly asks to skip the review ("push without review", "just push it")
- Pushing tags only (`git push --tags` with no commit changes)
- Force-pushing to recover from a known state (user has already reviewed)

## Rules

### 1. Review before every push

Before executing `git push`, run a code review on the changes that will be pushed:

```bash
# Determine what will be pushed
git log --oneline @{upstream}..HEAD 2>/dev/null || git log --oneline origin/$(git branch --show-current)..HEAD

# Run the active AI coding assistant's review command
# (e.g., /review in Copilot CLI, Claude Code, or equivalent)
```

Use whatever review command is available in your current environment. The specific command varies by platform (`/review`, `copilot review`, etc.).

### 2. Fix issues found by the review

If the review identifies problems:

1. **Read each issue** — understand what the reviewer flagged.
2. **Apply fixes** — edit the code to address valid concerns.
3. **Commit the fixes** — use the `git-commit` skill with a descriptive conventional commit message.

```bash
# Example fix commit
git add <fixed-files>
git commit -m "fix(<scope>): <description of what the review caught>"
```

### 3. Re-review until clean

After committing fixes, run the review again. Repeat the fix-commit-review cycle until the review passes with no issues.

```
Loop:
  1. Run review
  2. Issues found? → fix, commit, go to 1
  3. No issues? → proceed to push
```

Do not push with known unresolved review issues unless the user explicitly decides to override after seeing them.

### 4. Push only after a clean review

Once the review passes clean:

```bash
git push
```

If the branch has no upstream, set it:

```bash
git push -u origin $(git branch --show-current)
```

### 5. Report the result

Capture the commit range **before** pushing (after push, `HEAD` equals `@{upstream}` so the range is empty). Then report what was pushed:

```bash
# Before pushing — capture the range
PUSH_BASE=$(git rev-parse @{upstream} 2>/dev/null || git merge-base HEAD origin/HEAD 2>/dev/null)

# After pushing — show what was pushed
git log --oneline "$PUSH_BASE"..HEAD
```

## Full Recipe

```bash
# 1. Check what will be pushed
BRANCH=$(git branch --show-current)
UPSTREAM=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null)

if [ -z "$UPSTREAM" ]; then
  echo "No upstream set — will push with -u origin/$BRANCH"
  DEFAULT_BRANCH=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null)
  if [ -n "$DEFAULT_BRANCH" ]; then
    BASE=$(git merge-base HEAD "$DEFAULT_BRANCH")
    COMMITS=$(git log --oneline "$BASE"..HEAD)
  else
    COMMITS=$(git log --oneline -n 10 HEAD)
    echo "(no upstream or default branch found — showing last 10 commits)"
  fi
else
  COMMITS=$(git log --oneline "$UPSTREAM"..HEAD)
fi

echo "Commits to push:"
echo "$COMMITS"

# 2. Run review (repeat until clean)
# Use the review command available in your environment.
# The loop below uses the exit code to determine pass/fail.
CLEAN=false
while [ "$CLEAN" = false ]; do
  if <review-command>; then
    CLEAN=true
  else
    echo "Review found issues. Fix them, commit the changes, and rerun."
    # ... apply fixes, git add, git commit ...
  fi
done

# 3. Push
if [ -z "$UPSTREAM" ]; then
  git push -u origin "$BRANCH"
else
  git push
fi
```

## Examples

```bash
# BAD — pushing without review
git push

# BAD — reviewing but pushing despite issues
<review-command>  # finds 3 issues
git push            # pushes anyway

# GOOD — review, fix, re-review, push
<review-command>  # finds 2 issues
# ... fix the issues ...
git add -A && git commit -m "fix(auth): address review findings in token validation"
<review-command>  # clean
git push
```

## Red Flags — STOP

- About to run `git push` without having run a review first → run the review
- Review found issues and you are about to push anyway → fix the issues first
- About to skip the re-review after committing fixes → run the review again to confirm
- Stuck in a fix-review loop (3+ iterations) → present the remaining issues to the user and ask how to proceed
