---
name: redact-secrets
description: Detect and redact secrets in text. Use when sharing logs, snippets, or pastes that may contain API keys, OAuth tokens, JWTs, AWS credentials, GitHub tokens, Slack tokens, database connection strings, or private keys.
---

# Redact secrets

When the user asks you to share, summarise, or paste any block of text — logs, code, env files, terminal output, error messages, config files, network requests — first scan it for secrets and replace them before producing your output.

## What counts as a secret

Treat any of these as a secret:

- **API keys / tokens**: long random strings prefixed by a known vendor scheme (`sk-`, `xoxb-`, `xoxp-`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `github_pat_`, `glpat-`, `AIza`, `AKIA`)
- **JWTs**: three base64url segments separated by `.`, where the first segment decodes to JSON containing `alg`
- **AWS credentials**: `AKIA[0-9A-Z]{16}` access key IDs and the 40-character secret access keys that accompany them
- **OAuth bearer tokens**: anything passed in `Authorization: Bearer …`
- **Connection strings**: `postgres://user:password@host/db`, `mongodb+srv://…`, `mysql://…` — replace the password component, keep the rest
- **Private keys**: anything between `-----BEGIN … PRIVATE KEY-----` and `-----END … PRIVATE KEY-----`
- **Generic high-entropy strings**: ≥32 chars of mixed case + digits + symbols, where the surrounding context (variable name, comment, key name) suggests a credential — `secret`, `token`, `key`, `password`, `credential`, `auth`

Don't redact: UUIDs, git commit SHAs, public URLs, hashes of public artefacts, well-known constants (zero-vectors, all-ones), or strings the user explicitly says are non-sensitive.

## How to redact

Replace the secret value with a placeholder that preserves enough shape to be obviously a redaction without leaking the original:

- Default placeholder: `[REDACTED]`
- For prefixed tokens, keep the prefix and the first 4 characters so the kind is identifiable: `ghp_AbCd[REDACTED]`, `sk-Pq3M[REDACTED]`
- For connection strings, redact only the password: `postgres://user:[REDACTED]@host/db`
- For private keys, replace the whole block with `-----BEGIN … PRIVATE KEY-----\n[REDACTED]\n-----END … PRIVATE KEY-----`
- Keep line counts roughly the same so log structure is preserved

Always redact in-place — don't reorder, summarise, or otherwise mutate the surrounding text.

## When you find none

If you scan and find nothing, output the original text unchanged. Don't add a "no secrets found" preamble — that pollutes pipeable output. The absence of redactions is the signal.

## When you're unsure

If a string looks suspicious but you can't classify it confidently (e.g., a 40-char hex string with no surrounding context), redact it and add a single trailing comment line: `# 1 ambiguous string redacted at line N`. Surface ambiguity rather than hide it.

## Examples

Input:
```
DATABASE_URL=postgres://app:hunter2@db.internal:5432/prod
GITHUB_TOKEN=ghp_AbCd1234efghIJKLmnop5678QRSTuvwxYZ0
# user_id is fine to share
USER_ID=018f1c2a-9d4b-7e91-aaaa-bbbbcccc1111
```

Output:
```
DATABASE_URL=postgres://app:[REDACTED]@db.internal:5432/prod
GITHUB_TOKEN=ghp_AbCd[REDACTED]
# user_id is fine to share
USER_ID=018f1c2a-9d4b-7e91-aaaa-bbbbcccc1111
```
