---
name: git-push
description: "Use when pushing changes to a remote repository - spawns a Copilot CLI session to run /review before pushing, fixes any issues found, and repeats until the review passes clean"
---

# Git Push with Pre-Push Review

## Overview

Before pushing changes to a remote repository, spawn a **Copilot CLI** session and run its `/review` slash command. This catches problems before they reach the remote. Fix any issues found, commit the fixes, and re-run `/review` in Copilot CLI until clean. Only then push.

**Important:** The review MUST be performed by Copilot CLI (`/review`), not by the current AI coding assistant. This ensures an independent review regardless of which agent is driving the session.

## When to Use

- Running `git push` to any remote branch
- Pushing after committing fixes, features, or refactors
- Any workflow step that sends local commits to a remote

## Do NOT Use When

- Pushing tags only (`git push --tags` with no commit changes)
- Force-pushing to recover from a known state (user has already reviewed)

## Rules

### 1. Run Copilot CLI /review before every push

Before executing `git push`, spawn a Copilot CLI session and run `/review`:

```bash
copilot
# then inside the Copilot CLI session:
/review
```

If `/review` reports issues, do **not** push yet.

### 2. Fix issues found by /review

For each issue flagged:

1. Read and understand the issue.
2. Apply the fix.
3. Commit using the `git-commit` skill.

### 3. Re-run /review until clean

After committing fixes, run `/review` in Copilot CLI again. Repeat the fix-commit-review cycle until it passes with no issues.

If stuck after 3+ iterations, present the remaining issues to the user and ask how to proceed.

### 4. Push only after a clean /review

```bash
git push
```

### 5. If the user asks to skip /review

The review MUST still run. Present the findings and require explicit acknowledgement before pushing with known issues.

## Examples

```bash
# BAD — pushing without review
git push

# GOOD — Copilot CLI /review, fix, re-review, push
copilot              # spawn Copilot CLI
/review              # finds 2 issues
# ... fix the issues ...
git add <files> && git commit -m "fix(auth): address review findings"
/review              # clean
git push
```

## Red Flags — STOP

- About to run `git push` without having run Copilot CLI `/review` first → run it
- `/review` found issues and you are about to push anyway → fix them first
- About to skip re-running `/review` after committing fixes → run it again
