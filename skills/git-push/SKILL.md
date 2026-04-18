---
name: git-push
description: "Use when pushing changes to a remote repository - runs GitHub Copilot review before pushing, fixes any issues found, and repeats until the review passes clean"
---

# Git Push with Pre-Push Review

## Overview

Before pushing changes to a remote repository, run a **GitHub Copilot** code review first. This catches problems before they reach the remote. Fix any issues found, commit the fixes, and re-run the Copilot review until clean. Only then push.

**Important:** The review MUST be performed by GitHub Copilot, not by the current AI coding assistant. Use `gh copilot review` or the Copilot CLI to run the review.

## When to Use

- Running `git push` to any remote branch
- Pushing after committing fixes, features, or refactors
- Any workflow step that sends local commits to a remote

## Do NOT Use When

- Pushing tags only (`git push --tags` with no commit changes)
- Force-pushing to recover from a known state (user has already reviewed)

## Rules

### 1. Run Copilot review before every push

Before executing `git push`, run `Copilot review` to inspect the changes:

```bash
gh copilot review
```

If `Copilot review` reports issues, do **not** push yet.

### 2. Fix issues found by Copilot review

For each issue flagged:

1. Read and understand the issue.
2. Apply the fix.
3. Commit using the `git-commit` skill.

### 3. Re-run Copilot review until clean

After committing fixes, run `Copilot review` again. Repeat the fix-commit-review cycle until `Copilot review` passes with no issues.

If stuck after 3+ iterations, present the remaining issues to the user and ask how to proceed.

### 4. Push only after a clean Copilot review

```bash
git push
```

### 5. If the user asks to skip Copilot review

The review MUST still run. Present the findings and require explicit acknowledgement before pushing with known issues.

## Examples

```bash
# BAD — pushing without Copilot review
git push

# GOOD — review, fix, re-review, push
gh copilot review        # finds 2 issues
# ... fix the issues ...
git add <files> && git commit -m "fix(auth): address review findings"
gh copilot review        # clean
git push
```

## Red Flags — STOP

- About to run `git push` without having run `Copilot review` first → run it
- `Copilot review` found issues and you are about to push anyway → fix them first
- About to skip re-running `Copilot review` after committing fixes → run it again
