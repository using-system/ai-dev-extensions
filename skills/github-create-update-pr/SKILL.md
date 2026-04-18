---
name: github-create-update-pr
description: "Use when creating or updating GitHub pull requests - enforces Conventional Commits naming convention on PR titles for semantic versioning and squash-merge compatibility, and verifies that CI workflows (if any) complete successfully before handing the PR back to the user"
---

# GitHub Create/Update PR Convention

## Overview

All GitHub pull request titles MUST follow the [Conventional Commits](https://www.conventionalcommits.org/) specification. When a repo uses squash-merge, the PR title becomes the commit message, so it must be a valid conventional commit to enable automatic semantic versioning and changelog generation.

## When to Use

- Creating a new pull request
- Updating an existing PR title or body
- When a user asks about PR naming conventions

## Format

PR title follows the same format as a conventional commit:

```
<type>(<scope>): <description>
```

### Types

| Type | Description | SemVer Impact |
|------|-------------|---------------|
| `feat` | New feature | MINOR |
| `fix` | Bug fix | PATCH |
| `docs` | Documentation only | - |
| `style` | Formatting, no code change | - |
| `refactor` | Code restructuring, no behavior change | - |
| `perf` | Performance improvement | PATCH |
| `test` | Adding or correcting tests | - |
| `build` | Build system or dependency changes | - |
| `ci` | CI configuration changes | - |
| `chore` | Maintenance, no src/test change | - |
| `revert` | Reverts a previous change | - |

### Breaking Changes

Indicate breaking changes with `!` after the type/scope:

```
feat(api)!: remove deprecated v1 endpoints
```

## Rules

### 1. PR title must be a valid conventional commit

```
# BAD
Add login page
Feature: login page
[FEAT] Add login page

# GOOD
feat: add login page
feat(auth): add login page
```

### 2. Same rules as git-commit for the title line

- Lowercase description
- Imperative mood ("add" not "added" or "adds")
- No period at the end
- Type is mandatory, scope is optional but encouraged

### 3. PR body should complement the title

The PR body is free-form but should include:
- **What**: summary of changes
- **Why**: motivation and context
- **How to test**: steps for reviewers

### 4. One logical change per PR

A PR should map to a single conventional commit type. If a PR contains both a feature and a refactor, the dominant change defines the type.

### 5. Branch name should align with PR title

```
# PR title: feat(auth): add OAuth2 login flow
# Branch name:
feat/auth-oauth2-login-flow

# PR title: fix(api): handle null payment response  
# Branch name:
fix/api-null-payment-response
```

### 6. Verify CI workflows after creating or updating the PR

After every `gh pr create` or push that updates an open PR, check whether the repository has CI workflows that run on `pull_request` (or equivalent) and confirm they pass before handing control back to the user. Do not declare the PR ready if checks are red or still running.

#### Detect whether the repo has CI

```bash
# Any workflows defined?
ls .github/workflows 2>/dev/null

# What checks are expected on this PR?
gh pr checks <pr-number-or-branch>
```

If `.github/workflows/` is absent or empty **and** `gh pr checks` reports no checks, skip the rest of this rule — there is nothing to verify.

#### Wait for checks to complete

```bash
gh pr checks <pr-number-or-branch> --watch --fail-fast
```

`--watch` streams status until every check reaches a terminal state; `--fail-fast` returns a non-zero exit code as soon as one check fails so you can react immediately.

#### If checks pass

Report success to the user with the PR URL. Done.

#### If checks fail

1. Identify the failing job(s):

   ```bash
   gh pr checks <pr-number-or-branch>
   gh run list --branch <branch> --limit 5
   ```

2. Fetch the failing logs:

   ```bash
   gh run view <run-id> --log-failed
   ```

3. Diagnose the root cause before reacting. Do **not** blindly add retries, `continue-on-error`, or workflow edits just to make the red turn green — that hides real defects. Common categories:

   - **Code defect in the PR** → fix the code, commit, push, re-verify.
   - **Broken workflow (misconfigured action, missing secret, wrong version pin)** → fix the workflow per the `github-actions` skill, commit, push, re-verify.
   - **Pre-existing failure unrelated to this PR** → stop, surface it to the user with the log evidence, and let them decide whether to fix here or in a separate PR.
   - **Flake** → rerun once via `gh run rerun <run-id>`. If it fails again, treat as a real defect.

4. After any fix, push and re-run `gh pr checks --watch --fail-fast` to confirm green.

#### Never merge around red CI

If the user asks you to merge while checks are failing, refuse and explain the failing checks. Merging past red CI is a destructive action — require explicit, informed authorization.

## Examples

```
# Feature
feat(dashboard): add real-time metrics widget

# Bug fix
fix(auth): prevent token refresh race condition

# Documentation
docs(readme): add installation guide

# CI improvement
ci(workflows): pin GitHub Actions to SHA

# Breaking change
feat(api)!: migrate from REST to GraphQL
```
