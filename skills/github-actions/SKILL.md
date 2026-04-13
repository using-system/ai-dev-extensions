---
name: github-actions
description: "Use when reviewing, writing, or modifying GitHub Actions workflows or referencing any GitHub Action - enforces version pinning by commit SHA, latest version verification, version commenting, and script injection prevention"
---

# GitHub Actions Best Practices

## Overview

Every GitHub Action reference in a workflow MUST be pinned to a full commit SHA (not a tag or branch) with the human-readable version noted in an inline comment. Before writing or reviewing any workflow, always verify that the latest version of each action is being used.

## When to Use

- Writing a new GitHub Actions workflow (`.github/workflows/*.yml`)
- Reviewing or modifying an existing workflow
- Adding a new action (`uses:` directive) to a workflow
- Auditing workflow security
- When a user asks about GitHub Actions best practices

## Rules

### 1. Always pin actions to full commit SHA

**Never** reference actions by tag or branch:

```yaml
# BAD - tag reference, vulnerable to tag mutation
- uses: actions/checkout@v4

# BAD - branch reference, non-deterministic
- uses: actions/checkout@main

# GOOD - full SHA pin with version comment
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

### 2. Always verify the latest version is available

Before writing or approving any action reference:

1. Check the action's repository for the latest release
2. Get the full commit SHA for that release tag
3. Use the SHA, not the tag

To verify a SHA for a given tag:
```bash
git ls-remote --tags https://github.com/{owner}/{repo}.git refs/tags/{tag}
```

Or via GitHub API:
```bash
gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
gh api repos/{owner}/{repo}/git/ref/tags/{tag} --jq '.object.sha'
```

### 3. Always add a version comment

The inline comment after the SHA MUST contain the human-readable version so that:
- Humans can quickly see which version is used
- Dependabot / Renovate can still detect updates
- Diffs remain reviewable

Format: `{owner}/{action}@{sha} # {tag}`

### 4. Never use expressions directly in `run` scripts — use environment variables

Interpolating `${{ }}` expressions directly in `run:` blocks (bash, PowerShell, or any shell) creates **script injection vulnerabilities**. An attacker who controls the expression value (e.g., a PR title, branch name, or issue body) can inject arbitrary commands.

**Always** pass untrusted values through environment variables:

```yaml
# BAD - direct interpolation, vulnerable to command injection
- run: echo "Hello ${{ github.event.pull_request.title }}"

# BAD - same issue in PowerShell
- run: Write-Host "Hello ${{ github.event.issue.body }}"
  shell: pwsh

# GOOD - pass through env, shell handles escaping
- run: echo "Hello $TITLE"
  env:
    TITLE: ${{ github.event.pull_request.title }}

# GOOD - PowerShell equivalent
- run: Write-Host "Hello $env:BODY"
  shell: pwsh
  env:
    BODY: ${{ github.event.issue.body }}
```

#### Dangerous expression sources

These expressions are **user-controlled** and MUST always go through `env:`:

| Source | Example |
|--------|---------|
| PR title / body | `github.event.pull_request.title`, `github.event.pull_request.body` |
| Issue title / body | `github.event.issue.title`, `github.event.issue.body` |
| Comment body | `github.event.comment.body` |
| Commit message | `github.event.head_commit.message` |
| Branch / tag name | `github.head_ref`, `github.ref_name` |
| Review body | `github.event.review.body` |

#### Safe expressions (env still recommended)

These are generally safe but using `env:` is still best practice for consistency:

- `github.repository`, `github.actor`, `github.sha`
- `github.run_id`, `github.run_number`
- `secrets.*`, `vars.*`
- `needs.<job>.outputs.*` (if outputs are sanitized)

### 5. Apply to ALL actions, including official ones

This applies to **every** action, including:
- `actions/checkout`
- `actions/setup-node`
- `actions/cache`
- `docker/login-action`
- Third-party and marketplace actions

No exceptions.

## Why This Matters

- **Supply chain security**: Tags are mutable. An attacker who compromises a repo can move a tag to point at malicious code. SHA pins are immutable.
- **Reproducibility**: SHA ensures the exact same code runs every time.
- **Auditability**: The version comment makes it easy to track what's deployed without resolving SHAs manually.

## Review Checklist

When reviewing a workflow PR, verify each `uses:` line:

- [ ] Pinned to full 40-character commit SHA
- [ ] SHA corresponds to the **latest** available version
- [ ] Inline comment shows the version tag (e.g., `# v4.2.2`)
- [ ] If not latest, there is a documented reason (compatibility, etc.)

And verify each `run:` block:

- [ ] No `${{ }}` expressions interpolated directly in the script
- [ ] User-controlled values (PR title, issue body, branch name, etc.) passed through `env:`
- [ ] Environment variables used in the shell script instead of inline expressions

## Common Actions Reference

Keep these up to date. When writing workflows, always verify current latest versions:

| Action | Check latest |
|--------|-------------|
| `actions/checkout` | `gh api repos/actions/checkout/releases/latest --jq '.tag_name'` |
| `actions/setup-node` | `gh api repos/actions/setup-node/releases/latest --jq '.tag_name'` |
| `actions/setup-python` | `gh api repos/actions/setup-python/releases/latest --jq '.tag_name'` |
| `actions/cache` | `gh api repos/actions/cache/releases/latest --jq '.tag_name'` |
| `docker/build-push-action` | `gh api repos/docker/build-push-action/releases/latest --jq '.tag_name'` |
| `docker/login-action` | `gh api repos/docker/login-action/releases/latest --jq '.tag_name'` |
| `azure/login` | `gh api repos/azure/login/releases/latest --jq '.tag_name'` |
| `aws-actions/configure-aws-credentials` | `gh api repos/aws-actions/configure-aws-credentials/releases/latest --jq '.tag_name'` |
| `google-github-actions/auth` | `gh api repos/google-github-actions/auth/releases/latest --jq '.tag_name'` |
