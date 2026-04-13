# ai-dev-extensions

A curated collection of personal agent extensions, skills, and CLI hooks to enhance AI coding agents like Claude Code, Cursor, Codex, OpenCode, Copilot, and Gemini.

## Skills

| Skill | Description |
|-------|-------------|
| `github-actions` | Enforces GitHub Actions best practices: pin actions to SHA, verify latest versions, add version comments |
| `git-commit` | Enforces Conventional Commits specification for semantic versioning compatibility |
| `github-pr` | Enforces Conventional Commits naming on PR titles for squash-merge and changelog generation |

## Installation

### Claude Code

```bash
claude plugin marketplace add using-system/ai-dev-extensions
claude plugin install ai-dev-extensions@using-system
```

Or for a single session from a local clone:

```bash
claude --plugin-dir /path/to/ai-dev-extensions
```

### Cursor

```bash
/plugin marketplace add using-system/ai-dev-extensions
/plugin install ai-dev-extensions@using-system
```

### Copilot CLI

```bash
copilot plugin marketplace add using-system/ai-dev-extensions
copilot plugin install ai-dev-extensions@using-system
```

### Codex

```bash
git clone https://github.com/using-system/ai-dev-extensions.git ~/.codex/ai-dev-extensions
mkdir -p ~/.agents/skills
ln -s ~/.codex/ai-dev-extensions/skills ~/.agents/skills/ai-dev-extensions
```

See [docs/README.codex.md](docs/README.codex.md) for details.

### OpenCode

Add to your `opencode.json`:

```json
{
  "plugin": ["ai-dev-extensions@git+https://github.com/using-system/ai-dev-extensions.git"]
}
```

See [docs/README.opencode.md](docs/README.opencode.md) for details.

### Gemini CLI

```bash
gemini extensions install https://github.com/using-system/ai-dev-extensions
```

## Structure

```
ai-dev-extensions/
├── skills/                    # Skills (one directory per skill)
│   ├── github-actions/        # GitHub Actions best practices
│   │   └── SKILL.md
│   ├── git-commit/            # Conventional Commits for git
│   │   └── SKILL.md
│   └── github-pr/             # Conventional Commits for PR titles
│       └── SKILL.md
├── hooks/                     # Hooks for CLI agents and git
│   ├── session-start          # Session start hook (multi-platform)
│   ├── commit-msg             # Git hook: validates Conventional Commits
│   ├── install-hooks          # Install git hooks into any repo
│   ├── hooks.json             # Claude Code plugin hooks
│   └── hooks-cursor.json      # Cursor plugin hooks
├── docs/                      # Platform-specific install guides
├── .claude-plugin/            # Claude Code plugin manifest
├── .cursor-plugin/            # Cursor plugin manifest
└── CLAUDE.md                  # Contributor guidelines
```

## Git Hooks

Install the git hooks (Conventional Commits validation) into any repo:

```bash
/path/to/ai-dev-extensions/hooks/install-hooks /path/to/your/repo
```

This symlinks `commit-msg` into your repo's `.git/hooks/`. Works for all agents and humans alike.

## License

Apache-2.0
