---
name: safe-commit
description: Methodical commit workflow with security checks and change classification. Use when user says "commit this", "safe commit", "commit and log", or after completing a significant code change.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: workflow
  tags: [git, commit, security, process]
---

# Safe Commit

A disciplined commit workflow that ensures every code change has: a security check, well-classified commit message, and verification.

## Instructions

### Step 1: Review changes

```bash
git status
git diff --staged
git diff
```

Understand what changed. Classify the change type:
- **feat**: New feature or capability
- **fix**: Bug fix
- **refactor**: Code restructuring without behavior change
- **config**: Configuration, environment, or dependency change
- **ui**: Frontend, rendering, or display change
- **lm**: LLM prompt, template, or AI-related change
- **docs**: Documentation update

### Step 2: Security check

Before committing, verify:
- [ ] No secrets in staged files (.env, API keys, tokens, passwords)
- [ ] No hardcoded credentials
- [ ] No SQL injection vectors in new database queries
- [ ] No XSS vectors in new HTML rendering
- [ ] No command injection in new shell calls
- [ ] Dependencies being added are from trusted sources

If any security issue is found, **fix it before proceeding**. Do not commit insecure code.

### Step 3: Stage and commit

Stage specific files (never `git add -A` or `git add .`):
```bash
git add <specific-files>
```

Write a descriptive commit message using the type prefix:
```bash
git commit -m "$(cat <<'EOF'
<type>: <concise description>

<optional body explaining why, not what>
EOF
)"
```

### Step 4: Verify

```bash
git status
git log --oneline -3
```

Confirm the commit succeeded and looks correct.

## Examples

### Example 1: Feature commit
```
feat: add research gap scanning for any domain

Searches arxiv + Semantic Scholar, extracts future work sections,
validates gaps are confirmed unsolved.
```

### Example 2: Fix commit
```
fix: advice response logging when user only asked a question

Was triggering journal capture on every advice request instead of
only when user explicitly asked to log something.
```

### Example 3: LLM change
```
lm: tighten advice word budgets in ADVICE_TMPL

Simple: 60-100 words, Medium: 100-150, Complex: 150-200.
Hard cap 200. No filler openers.
```
