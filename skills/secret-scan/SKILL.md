---
name: secret-scan
description: Scans staged or committed changes for leaked secrets (Stripe keys, .env files, API tokens, private keys). Auto-invoke before commit, before PR creation, or when user mentions secret, leak, credential, or API key.
model: haiku
effort: low
tools: Bash, Grep, Read
color: red
---
# Secret Scan

Pre-commit leak detection. Cheap insurance before any push.

## When to run

- Before `git commit` (pre-commit check)
- Before `gh pr create` (pre-PR check)
- On demand when the user mentions secrets, leaks, or credentials
- As part of the `deploy-checklist` before production deploys

## What to scan

Run grep over staged changes and the full working tree. Patterns to catch:

### Stripe
- `sk_live_[a-zA-Z0-9]{24,}` — live secret key
- `sk_test_[a-zA-Z0-9]{24,}` — test secret key (lower severity, still flag)
- `rk_live_[a-zA-Z0-9]{24,}` — live restricted key
- `whsec_[a-zA-Z0-9]{32,}` — webhook signing secret

### OpenAI / Anthropic
- `sk-[a-zA-Z0-9]{48,}` — OpenAI API key
- `sk-ant-[a-zA-Z0-9-]{90,}` — Anthropic API key

### Generic
- `[a-zA-Z0-9_-]*(API_KEY|SECRET|TOKEN|PASSWORD)\s*=\s*['\"][^'\"]{16,}['\"]`
- `-----BEGIN (RSA |EC |DSA |OPENSSH |)PRIVATE KEY-----` — private keys
- `ghp_[a-zA-Z0-9]{36}` / `gho_[a-zA-Z0-9]{36}` — GitHub tokens
- `xoxb-[0-9]{11,}-[0-9]{11,}-[a-zA-Z0-9]{24}` — Slack bot token

### File-based leaks
- Any `.env` file that is not `.env.example` appearing in staged changes
- Any `.pem`, `.key`, `id_rsa` files in staged changes

## Scan commands

```bash
# Scan staged changes
git diff --cached | grep -E '(sk_live_|sk_test_|sk-ant-|-----BEGIN .*PRIVATE KEY-----|API_KEY\s*=\s*["'\''])'

# Check for .env files being committed
git diff --cached --name-only | grep -E '^\.env($|\.)|^[^/]*/\.env($|\.)' | grep -v '\.env\.example$'

# Deep scan the working tree (slower, use when asked)
git grep -nE '(sk_live_|sk_test_|sk-ant-|-----BEGIN .*PRIVATE KEY-----)'
```

## Output format

If clean:
```
✅ Secret scan: clean (staged changes)
```

If findings:
```
🔴 Secret scan: 2 findings

1. src/lib/stripe.ts:12 — matches sk_live_ pattern
   → REMOVE before commit. Use STRIPE_SECRET_KEY env var.

2. .env staged for commit
   → REMOVE from git with `git rm --cached .env`. Ensure .env is in .gitignore.
```

## Remediation

If a secret has already been committed:
1. Rotate the secret immediately (revoke + reissue)
2. Rewrite history with `git filter-repo` or BFG if the commit hasn't been pushed
3. If already pushed, rotate is the only safe fix — GitHub scrubbing does not help for anything that was publicly visible

Never tell the user "just squash it" — anyone who cloned before the squash still has the secret.

## False positives

Test fixtures using obvious dummy values (`sk_test_00000...`, `API_KEY=example`) are fine — flag them as 🟡 low severity but don't block.
