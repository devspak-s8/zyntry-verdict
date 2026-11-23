# Security Audit Report: verdict.zyntry.space

**Date:** Nov 18, 2026 · **Scope:** Worker gateway, D1, Container, CLI, Action
**Status:** ✅ All automated checks passed (20/20)

## What was tested

### Authentication & authorization
- No key → 401 ✓
- Fake key → 401 ✓
- Path traversal in key header → 401 ✓
- Cross-tenant verdict access → 404 ✓ (IDOR blocked)
- Unknown topup reference → 404 ✓ (IDOR blocked)
- Provider key never returned in GET ✓ (write-only storage)

### Rate limiting & anti-abuse
- Signup: per-IP daily limit (5/day) → 429 ✓
- Trial credits: $1 (reduced from $5 to prevent farming) ✓
- Verdicts: per-key per-minute limit → 429 ✓
- Oversized body (6MB) → rejected ✓
- Malformed JSON → 4xx ✓

### Injection
All SQL injection probes (DROP TABLE, OR 1=1, stacked queries, XSS, path traversal) → 4xx, no 500s. All D1 queries use parameterized `.bind()`.

### Webhook security
- HMAC-SHA256 signature verification (`X-Bachs-Signature-V2`)
- 300-second replay tolerance (stale + future timestamps rejected)
- Tampered body → signature fails (byte-for-byte coverage)
- Wrong secret → fails
- Rotation: any valid `v1=` accepted
- 8/8 unit tests pass (`packages/worker/test/bachs-webhook.test.ts`)
- Live forged delivery → rejected ✓ (503 until BACHS_WEBHOOK_SECRET is set, then 400)

### Secrets hygiene
- Full git history scan (all branches, all commits): **0 findings** ✓
- `.env` never committed ✓
- META_API_KEY (LLM_...) never in any commit ✓
- `.gitignore` covers `.env`, `vendor/`, `gateway-data/`, `.wrangler/`, personal videos ✓
- Provider keys: AES-GCM encrypted at rest via `ZV_ENC_KEY` secret, decrypted in-memory only

### Headers
- `X-Content-Type-Options: nosniff` ✓
- `Cache-Control: no-store` ✓
- `Referrer-Policy: no-referrer` ✓
- HSTS: set at Cloudflare zone level

### Container hardening
- **Non-root user** (`USER node` in Dockerfile) ✓
- **Wall-clock kill**: 15-minute hard cap per verdict, then the container is signalled and the verdict recorded as error ✓
- **Egress allowlist** (`ZV_EGRESS_ALLOW`): model API host must be allowlisted; http tool targets also gated ✓
- **Sealed standards**: physically staged outside the agent sandbox ✓

## What was fixed during this audit

| # | Gap | Fix | Status |
|---|---|---|---|
| 1 | Trial farming ($5 × unlimited signups) | $1 trial + per-IP daily signup limit (5/day) | ✅ deployed, pentested |
| 2 | Container egress not enforced | `ZV_EGRESS_ALLOW` allowlist + http tool gate | ✅ deployed |
| 3 | CLI key stored plaintext | `chmod 0600` on config file | ✅ shipped |
| 4 | Dockerfile ran as root | `USER node` | ✅ deployed |
| 5 | No wall-clock kill on containers | 15-min timer in DO → `container.signal()` | ✅ deployed |
| 6 | No signup rate limit | D1 counter per IP/day, limit 5 | ✅ deployed, pentested |
| 7 | No verdict rate limit | D1 counter per key/minute, limit 6 | ✅ deployed |
| 8 | No security headers | `nosniff`, `no-store`, `no-referrer` on all API responses | ✅ deployed |
| 9 | CaseId injection → charged + streamed as error | Regex validate caseId against `INSTALLED_CASES` before charging | ✅ deployed |
| 10 | Webhook verification untested | 8 unit tests (tamper, replay, wrong secret, rotation) | ✅ 8/8 |

## Known remaining items (documented, not blocking launch)

1. **Bachs webhook secret not yet set**: returns 503 on all deliveries (rejects everything, fail-closed). Once the signing secret is set, forged signatures get 400.
2. **ZV_ENC_KEY rotation runbook**: no re-encryption path yet; documented as an operational procedure (rotate = set new secret, all stored provider keys become undecryptable, users re-enter). Acceptable for v1.
3. **OS-tier container egress-deny**: platform-level network policy (default-deny namespace); app-tier allowlist is enforced. Tracked in HOSTED_SANDBOX.md.
4. **npm account 2FA**: required before publishing `zyntry-verdict` to npm.

## How to re-run

```bash
# unit tests (webhook forgery)
node --experimental-strip-types --test packages/worker/test/bachs-webhook.test.ts

# secrets scan (full git history)
node scripts/security/secrets-scan.mjs

# live pentest (against production)
node scripts/security/pentest.mjs
```