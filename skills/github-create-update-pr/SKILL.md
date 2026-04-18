---
name: github-create-update-pr
description: "Use when creating or updating GitHub pull requests - enforces Conventional Commits naming convention on PR titles for semantic versioning and squash-merge compatibility"
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
