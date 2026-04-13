# ai-dev-extensions

Personal collection of skills, hooks and extensions for AI dev CLI agents.

## Conventions (eat your own dog food)

This repo MUST follow all its own skills. Before performing any action, check if a relevant skill exists in `skills/*/SKILL.md` and apply it.

## Structure

- `skills/` - Each skill is a directory with a `SKILL.md` file (YAML frontmatter + markdown)
- `hooks/` - Session hooks for different platforms (Claude Code, Cursor, Copilot)
- `docs/` - Platform-specific installation guides

## Skill format

Skills follow the agentskills.io specification:
- `name`: lowercase, hyphenated
- `description`: starts with "Use when...", max 1024 chars
- Body: Overview, When to Use, Rules sections

## Adding a new skill

1. Create `skills/{skill-name}/SKILL.md` with frontmatter
2. Keep content actionable and rule-based
3. Include concrete examples (good/bad patterns)
