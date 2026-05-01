---
name: redact-secrets
description: Detect and redact secrets in text. Use when sharing logs, snippets, or pastes that may contain API keys, OAuth tokens, JWTs, AWS credentials, GitHub tokens, Slack tokens, database connection strings, or private keys.
---

# Redact secrets

When the user asks you to share, summarise, or paste any block of text — logs, code, env files, terminal output, error messages, config files, network requests — first scan it for secrets and replace them before producing your output.

## What counts as a secret

Treat any of these as a secret:

- **API keys / tokens by vendor prefix**:
  - GitHub: `ghp_`, `gho_`, `ghu_`, `ghs_`, `github_pat_`
  - GitLab: `glpat-`
  - Slack: `xoxb-`, `xoxp-`, `xapp-`, `xoxe.xoxb-`
  - OpenAI / Anthropic: `sk-`, `sk-ant-`
  - Stripe: `sk_live_`, `sk_test_`, `rk_live_`, `whsec_`
  - Google API keys: `AIza` (39 chars total)
  - npm: `npm_` (40 chars)
  - Cloudflare: `cfat-`, `cfsk-`
  - SendGrid: `SG.`
  - Twilio: `AC` followed by 32 hex chars (account SID is also sensitive in combination with the token)
- **JWTs**: three base64url segments separated by `.`, where the first segment decodes to JSON containing `alg`
- **AWS credentials**: `AKIA[0-9A-Z]{16}` access key IDs and the 40-character secret access keys that accompany them; treat session tokens prefixed `ASIA` the same way
- **Google service-account JSON**: any JSON containing `"type": "service_account"` and a `"private_key"` field — redact the whole block, not just the key field
- **OAuth bearer tokens**: anything passed in `Authorization: Bearer …` — redact the token portion, keep the `Bearer ` prefix
- **Basic-auth headers**: `Authorization: Basic <base64>` — redact the base64 segment
- **Connection strings**: `postgres://user:password@host/db`, `mongodb+srv://…`, `mysql://…`, `redis://…`, `amqp://…` — replace the password component, keep the rest
- **Private keys**: anything between `-----BEGIN … PRIVATE KEY-----` and `-----END … PRIVATE KEY-----` (RSA, EC, OPENSSH, PGP)
- **SSH known-hosts / authorized-keys lines** containing private fingerprints from the user's machine
- **Webhook secrets / signing keys**: high-entropy values next to keys named `signing_secret`, `webhook_secret`, `hmac_key`, `signature`
- **Generic high-entropy strings**: ≥32 chars of mixed case + digits + symbols, where the surrounding context (variable name, comment, key name, header name, JSON path) suggests a credential — `secret`, `token`, `key`, `password`, `credential`, `auth`, `api_key`, `access_token`, `refresh_token`

Don't redact: UUIDs, git commit SHAs, public URLs, hashes of public artefacts, well-known constants (zero-vectors, all-ones), public SSH host keys / fingerprints already published in DNS or on the host's website, or strings the user explicitly says are non-sensitive.

## Contextual cues that escalate suspicion

A short or low-entropy value that wouldn't normally trigger redaction should still be redacted when the surrounding context suggests it's a credential:

- Variable name contains `password`, `passwd`, `pwd`, `secret`, `token`, `key`, `apikey`, `credential`, `auth` (case-insensitive)
- It appears after `=` in a `.env` / `.envrc` / `.secrets` file, or under a `secrets:` / `credentials:` YAML key
- It's a value for an HTTP header named `Authorization`, `X-API-Key`, `X-Auth-Token`, `Cookie` (when the cookie is named `session`, `auth`, `jwt`, `token`)
- It's in a `Set-Cookie` header for a session-shaped cookie
- It's in a URL query parameter named `token`, `access_token`, `api_key`, `signature`, `sig`

When in doubt in these contexts, redact.

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

## Code blocks vs prose

When the secret appears inside a fenced code block or otherwise machine-readable region, redact in-place — don't rewrap, don't drop or add backticks, don't add prose annotations inside the block. When the secret appears in surrounding prose ("the API key is `sk-abc...`"), redact the value but keep the prose readable; an inline note like *"(redacted)"* in parentheses is acceptable here.

## Common mistakes to avoid

- **Don't replace the variable name too.** `GITHUB_TOKEN=[REDACTED]` is right; `[REDACTED]=[REDACTED]` is not.
- **Don't redact the same value twice with different placeholders.** If the same token appears 3 times in the input, every occurrence should reduce to the same `[REDACTED]` form so the reader can tell they refer to the same secret.
- **Don't try to "be helpful" by reconstructing what the secret would have been** ("looks like an OpenAI key starting with sk-XXX") — that defeats the purpose. The kind hint in the prefix is fine; anything beyond that leaks information.
- **Don't drop secrets from logs silently** by removing the line. Keeping the line with a redacted value preserves the log's structure and tells the reader where the secret was.
- **Don't redact values the user has explicitly opted out of.** If they say "the host db.internal is fine to share, just hide the password", honour that.

## Examples

### Env file

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

### HTTP request log

Input:
```
POST /v1/charges HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_live_FAKE_DEMO_NOT_A_REAL_KEY_123
Content-Type: application/x-www-form-urlencoded
Idempotency-Key: 87b2eaba-3f9c-4f2e-8b6e-19c6a2b71b2e

amount=2000&currency=usd
```

Output:
```
POST /v1/charges HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_live_FAKE[REDACTED]
Content-Type: application/x-www-form-urlencoded
Idempotency-Key: 87b2eaba-3f9c-4f2e-8b6e-19c6a2b71b2e

amount=2000&currency=usd
```

(Idempotency key is a UUID — not a secret. Authorization header value is.)

### JWT in a Set-Cookie

Input:
```
HTTP/1.1 200 OK
Set-Cookie: session=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImV4cCI6MTcyMDAwMDAwMH0.abc123signature; HttpOnly; Secure; Path=/
```

Output:
```
HTTP/1.1 200 OK
Set-Cookie: session=[REDACTED]; HttpOnly; Secure; Path=/
```

(Whole JWT redacted as one unit. Cookie attributes preserved.)
