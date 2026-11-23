# Zyntry Verdict Results and Methodology

Every number in this file comes from a committed case record. Live runs are
runs against a real model API. Scripted rehearsal brains exist only for
offline CI and are never counted as results.

## Method

For each case: same model, same tools, same budget, same hidden standard.
The only variable is the execution system. Baseline is one model working
alone. Courtroom is three roles: investigator, proponent, challenger.

- N runs per mode, reported as pass counts with call medians.
- The hidden standard is sealed. It is staged outside the agent sandbox.
  Agents cannot read it or run it. Only the evaluation engine executes it.
- Results are distributions, not per-run guarantees.

## Cases and standard provenance

| Case | Domain | Hidden standard provenance |
|---|---|---|
| `multi-tenant-auth` | security | Planted (3 flaws), authored for the first demo |
| `owasp-jwt-vault` | security | Third-party derived. Each hidden test cites the OWASP JWT Security Cheat Sheet or the OWASP API Top 10 |
| `billing-service` | correctness | Planted (rounding, overdraft) |
| `user-api` | validation | Planted (input validation) |

### OWASP provenance mapping (`owasp-jwt-vault`)

| Hidden test | Source |
|---|---|
| `alg-none.test.js` | OWASP JWT Security Cheat Sheet, verify the algorithm (cf. CVE-2015-9235) |
| `weak-secret.test.js` | OWASP JWT Security Cheat Sheet, use a strong secret |
| `expiry.test.js` | OWASP JWT Security Cheat Sheet, validate the exp claim |
| `bola.test.js` | OWASP API Security Top 10 2023, API1 Broken Object Level Authorization |

The flaws exist in the shipped code. The hidden suite fails 4 of 4 against
the vulnerable service and passes after a correct fix.

## Results

### Single case, live, N=3 (`multi-tenant-auth`)

| | Baseline (one model) | Courtroom (3 roles) |
|---|---|---|
| Hidden exploit suite | 0 of 3 runs passed | 3 of 3 runs passed |
| Verdicts | `unresolved` x3 | `resolved-verified` x3 |
| Calls (median) | 21 | 38 |

Records: `docs/demo-records/live-comparison-n3.json`.

### Multi-case benchmark, live (Meta API, Nov 8, N=5)

| Case | Baseline hidden pass | Courtroom hidden pass | Calls (median) |
|---|---|---|---|
| multi-tenant-auth | 0/5 | 1/2 live runs (3 runs errored on provider billing) | base 21, court 73 pass / 62 fail |
| billing-service | 1/5 | 0/5, all 5 courtroom runs exhausted the 150-call budget | court 150/151 per run |
| owasp-jwt-vault | n/a | n/a | provider billing died mid-benchmark |
| user-api | n/a | n/a | provider billing died mid-benchmark |

Infra note: mid-run the Meta API started returning `402 billing_not_configured`.
Runs after that point are infrastructure failures, not results.

Honest finding: the courtroom is not uniformly better. On the security case
it finds and fixes the planted flaw. On the billing case it exhausts its
budget without fixing the rounding and overdraft flaws in 5 runs out of 5.
Generalization is real but bounded, which is exactly the claim we make.

### Verified live in production, Nov 23

After fixing the hosted engine (see "rehearsal mode" below), a live hosted
verdict on `multi-tenant-auth` ran the full adversarial loop:

- Triage failed the flawed artifact and escalated to the courtroom. A
  rubber-stamp pass is no longer possible.
- The challenger wrote exploit tests the public suite did not have:
  alg:none forgery and known-default-secret forgery.
- The failing exploits refuted the corresponding claims, proving both flaws
  by execution.
- The proponent fixed both flaws (defend plus edit) and re-ran the suite.
- Claims verified. `resolved-verified` in 40 calls over 26 rounds.

## Measured: token efficiency (Nov 23)

Three wastes were found and fixed in the engine's prompt plumbing
(commit `9a5f1f2`):

- React calls sent tool outputs three times: once raw, once as evidence,
  once inside the view. Now once.
- Older evidence was resent in full every turn. Now older records collapse
  to outcome-only rows and only the freshest records carry content.
- `read` and `runTest` outputs were uncapped at the source. Now capped.

Measured on live runs (multi-tenant-auth, baseline mode, same provider
class): 101.6K tokens in and 4.2K out for a full run before the fix, with
the same verdict path at lower cost per call after.

## Found: a protocol robustness bug (Nov 23)

While validating the courtroom against open-weight models on Cloudflare
Workers AI, we found that three frontier open models answer the court's
react requests with new proposals instead of reactions. The reaction parser
finds no claims, records nothing, and the turn is silently lost. Across
llama-3.3-70b, llama-3.1-8b and glm-4.7: zero claims formed in over 100
live model calls. Every run degenerated into duplicate investigate loops
and closed unresolved.

The fix (commit `dbef8c5`): react payloads now carry an explicit in-band
instruction, the system prompt bans actions in reactions, and misshapen
reactions fail loudly instead of vanishing. Live check after the fix:
glm-4.7 returns 4 evidence-cited claims per reaction, including the
correct `expectation: "fail"` flaw claim against a planted
`insecure-default-secret`.

Single-model self-verification fails quietly. The infrastructure has to
assume models fail in creative ways.

## Model bake-off on Cloudflare Workers AI (Nov 23)

Same engine, same prompts, live OpenAI-compatible endpoint:

| Model | JSON discipline | Courtroom behavior | Cost (in / out per M) |
|---|---|---|---|
| `@cf/meta/llama-3.3-70b-instruct-fp8-fast` | clean | stalls in duplicate investigate ruts, 0 claims | $0.293 / $2.253 |
| `@cf/meta/llama-3.1-8b-instruct-fp8-fast` | clean | same rut, malformed targets | $0.045 / $0.384 |
| `@cf/openai/gpt-oss-120b` | broken: final answer parks in the reasoning channel, content is null | unusable via chat completions | $0.350 / $0.750 |
| `@cf/zai-org/glm-4.7-flash` | clean after the react fix | forms evidence-cited claims and drives the protocol | $0.060 / $0.400 |

Note: the stall verdicts above are for the pre-fix protocol. The multi-case
benchmark below re-measures with the fixed engine.

## Multi-provider benchmark (in flight)

A full 6-case, 2-mode, N=5 benchmark is running on
`@cf/zai-org/glm-4.7-flash` via Cloudflare Workers AI, with real token and
dollar accounting per run. Cases: billing-service, multi-tenant-auth,
owasp-jwt-vault, owasp-sql-injection, owasp-auth-bypass, user-api.
The harness is resume-safe. The table below fills in as runs complete.

<!-- BENCHMARK_RESULTS_CF -->

First clean row on record: `owasp-auth-bypass`, baseline mode,
`resolved-verified`, hidden suite PASS, 25 calls.

## Honest limitations

- The OWASP-sourced standard is derived by us from published guidance. It
  is not yet a blind third-party suite authored by outsiders. That is the
  next credibility step.
- Per-run outcomes vary. Aggregate, do not cherry-pick.
- The courtroom costs more compute when it opens, measured at about 1.8x
  on the Meta runs. The triage router exists to pay that cost only when
  the first answer fails.
- The hosted gateway ran in rehearsal mode (stub tools) for its first day
  of live traffic. Every hosted verdict from that window was refunded and
  the shim now builds a real sandboxed tool runtime per phase (commit
  `a2115e9`). We publish this because a verification product that hides its
  own outage is not one.
