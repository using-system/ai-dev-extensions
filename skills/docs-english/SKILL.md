---
name: docs-english
description: "Use when writing or editing documentation, code comments, docstrings, README files, CHANGELOG entries, PR descriptions, commit messages, or any committed textual artifact - enforces English as the sole language regardless of the language used in the conversation"
---

# Documentation Language

## Overview

All documentation and code artifacts committed to the repository MUST be written in English, even when the user is conversing in another language (French, Spanish, German, etc.). English is the lingua franca of software engineering and ensures maximum discoverability, collaboration, and long-term maintainability.

The conversation language is for interacting with the user. The artifact language is always English.

## When to Use

- Writing or editing any file intended to be committed
- Adding or updating code comments and docstrings (JSDoc, TSDoc, Python docstrings, Rustdoc, etc.)
- Writing README, CHANGELOG, CONTRIBUTING, or any `*.md` file
- Writing commit messages (already enforced by the `git-commit` skill, but reinforced here)
- Writing pull request titles and descriptions
- Writing issue descriptions, labels, and comments
- Writing release notes
- Naming variables, functions, classes, files, branches
- Writing log messages, error messages, and user-facing strings in code
- Writing configuration comments (YAML, TOML, JSON5 comments)

## Scope: What This Skill Does NOT Cover

This skill applies to **committed artifacts only**. It does NOT change:

- The language used when replying to the user in chat
- The language of the user's own source content (if translating or quoting the user)
- Localized string resources (i18n `.po`, `.json`, `.xliff` files) whose purpose is to hold non-English translations

## Rules

### 1. Write all documentation in English

Regardless of the conversation language, the output written to files MUST be in English.

```markdown
<!-- BAD (user asked in French, agent wrote README in French) -->
# Mon Projet
Ce projet est un outil de ligne de commande pour gérer les tâches.

<!-- GOOD -->
# My Project
This project is a command-line tool for managing tasks.
```

### 2. Code comments and docstrings in English

```python
# BAD
def calculer_total(articles):
    """Calcule le total des articles dans le panier."""
    ...

# GOOD
def calculate_total(items):
    """Calculate the total of items in the cart."""
    ...
```

```typescript
// BAD
// Récupère l'utilisateur depuis la base de données
function getUser(id: string) { ... }

// GOOD
// Fetch the user from the database
function getUser(id: string) { ... }
```

### 3. Identifiers in English

Variable names, function names, class names, file names, and branch names MUST be in English.

```javascript
// BAD
const utilisateurs = await db.fetchUsers();
function envoyerEmail(destinataire) { ... }

// GOOD
const users = await db.fetchUsers();
function sendEmail(recipient) { ... }
```

### 4. Commit messages, PR titles, and branch names in English

```bash
# BAD
git commit -m "feat: ajouter la page de connexion"
git checkout -b feat/page-connexion

# GOOD
git commit -m "feat: add login page"
git checkout -b feat/login-page
```

### 5. Log and error messages in English

Error messages are documentation for operators and future developers. They MUST be in English.

```go
// BAD
return fmt.Errorf("impossible de se connecter à la base de données: %w", err)

// GOOD
return fmt.Errorf("failed to connect to database: %w", err)
```

### 6. Translate user-provided content when needed

If the user describes a feature in French and asks you to document it, translate the description into clear, idiomatic English before writing.

```
User (in French): "Ajoute un README qui explique que l'outil synchronise les tâches"

# BAD: copy user's French verbatim into README
# GOOD: translate the intent into English
"This tool synchronizes tasks across devices."
```

### 7. Respond to the user in their language

This rule applies to artifacts, not conversation. Continue replying to the user in whatever language they used. Only what you write to files must be English.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Mirroring the user's language into the file because they wrote in French | Translate to English before writing |
| Mixing English code with non-English comments | Keep both in English |
| Using non-English identifiers "because the domain is French" | Use English identifiers; document domain terms in a glossary if needed |
| Writing commit messages in the user's spoken language | Always English per `git-commit` + this skill |
| Translating i18n resource files to English | Leave locale-specific files alone; they exist to hold translations |

## Red Flags — STOP and Rewrite in English

- You are about to write `# ` or `// ` followed by non-English text into a source file
- You are about to name a variable, function, or branch with non-English words
- You typed the user's prompt language into a `README.md`, `CHANGELOG.md`, or commit message
- You translated an English library's docs into another language inside a code comment

All of these mean: rewrite in English before saving.
