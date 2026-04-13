---
name: git-commit
description: "Use when creating git commits - enforces Conventional Commits specification for semantic versioning compatibility"
---

# Git Commit Convention

## Overview

All git commit messages MUST follow the [Conventional Commits](https://www.conventionalcommits.org/) specification. This enables automatic semantic versioning, changelog generation, and clear project history.

## When to Use

- Creating any git commit
- Reviewing commit messages in a PR
- Amending or rewording a commit message
- When a user asks about commit message best practices

## Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | Description | SemVer Impact |
|------|-------------|---------------|
| `feat` | New feature | MINOR |
| `fix` | Bug fix | PATCH |
| `docs` | Documentation only | - |
| `style` | Formatting, missing semicolons, etc. (no code change) | - |
| `refactor` | Code change that neither fixes a bug nor adds a feature | - |
| `perf` | Performance improvement | PATCH |
| `test` | Adding or correcting tests | - |
| `build` | Changes to build system or external dependencies | - |
| `ci` | Changes to CI configuration files and scripts | - |
| `chore` | Other changes that don't modify src or test files | - |
| `revert` | Reverts a previous commit | - |

### Breaking Changes

A breaking change MUST be indicated by:
- `!` after the type/scope: `feat!: remove deprecated API`
- Or a `BREAKING CHANGE:` footer in the commit body

Breaking changes trigger a **MAJOR** version bump.

## Rules

### 1. Type is mandatory

```bash
# BAD
git commit -m "add login page"

# GOOD
git commit -m "feat: add login page"
```

### 2. Scope is optional but encouraged

Scope should be a noun describing the section of the codebase:

```bash
# GOOD
git commit -m "feat(auth): add OAuth2 login flow"
git commit -m "fix(api): handle null response from payment gateway"
git commit -m "ci(github-actions): upgrade checkout action to v4"
```

### 3. Description must be lowercase, imperative, no period

```bash
# BAD
git commit -m "feat: Added login page."
git commit -m "feat: Adds login page"

# GOOD
git commit -m "feat: add login page"
```

### 4. Breaking changes must be explicit

```bash
# With ! marker
git commit -m "feat(api)!: change authentication endpoint response format"

# With footer
git commit -m "feat(api): change authentication endpoint response format

BREAKING CHANGE: response now returns a JWT token instead of session ID"
```

### 5. Body should explain the why, not the what

```bash
# GOOD
git commit -m "fix(parser): handle edge case with empty arrays

The parser crashed when receiving an empty array from the API.
Root cause was a missing null check in the deserialization step."
```

## Examples

```bash
# Feature
feat(dashboard): add real-time metrics widget

# Bug fix
fix(auth): prevent token refresh race condition

# Documentation
docs(readme): add installation instructions for OpenCode

# CI
ci(workflows): pin actions to SHA for supply chain security

# Breaking change
feat(api)!: migrate from REST to GraphQL endpoints

BREAKING CHANGE: all /api/v1/* endpoints are removed, use /graphql instead
```
