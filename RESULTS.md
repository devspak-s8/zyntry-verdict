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
| `infra-policy` | **infrastructure** | CIS Controls v8 derived: remote-admin restriction (Control 4), data protection (Control 3), least-privilege IAM (Control 6), modern TLS enforcement. An AI-generated production cloud config whose own sanity suite passes while the sealed standard catches SSH open to the internet, unencrypted storage, owner/editor service accounts, and unenforced HTTPS |
| `data-analysis` | **statistics** | Sealed statistical standard: averages over paying customers only, unrounded growth ratios, conversion ranked by rate not count. An AI-generated revenue module whose public checks pass while the standard catches denominator dilution (28.5% understatement), round-before-divide growth loss, and the segment-count trap |

The last two rows are the generalization made demonstrated rather than
claimed: the same courtroom, the same sealed-standard machinery, the same
adversarial loop, pointed at infrastructure configuration and statistical
analysis instead of code. The engine never touches code — it speaks actions,
evidence, claims and verdicts; the domains live in the tool surfaces and the
standards. AI-generated infrastructure and AI-generated analysis are now
under the same court.

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

### Verified live in production, Nov 23 (plumbing verified; model layer was rehearsal)

After fixing the hosted engine (see "rehearsal mode" below), a hosted
verdict on `multi-tenant-auth` verified the production path:

- Triage failed the flawed artifact and escalated to the courtroom. A
  rubber-stamp pass is no longer possible.
- The challenger wrote exploit tests the public suite did not have:
  alg:none forgery and known-default-secret forgery.
- The failing exploits refuted the corresponding claims, proving both flaws
  by execution.
- The proponent fixed both flaws (defend plus edit) and re-ran the suite.
- Claims verified. `resolved-verified` in 40 calls over 26 rounds.

Honest correction, added Nov 24: the test account that ran this had no
BYOK provider configured, so the model layer of that run was the scripted
rehearsal brain, not a live model. What the run genuinely verified: the
real sandboxed tool layer (files really read, tests really written and
executed), the escalation logic, the sealed hidden standard, and the verdict
recording path. What it did not verify: a live model through the hosted
path. The live-model hosted run fills in below when it closes.

<!-- HOSTED_LIVE_MODEL_VERIFICATION -->

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

## Found: the MCP surface deadlocking itself (Nov 28)

While smoke-testing the new evaluation tools against production, we found
that three of the original ten MCP tools returned `error code: 522` for
every call. `zyntry_get_verdict`, `zyntry_submit_run` and
`zyntry_verify_and_fix` all routed through a Worker subrequest to the
gateway's own public URL. Cloudflare can serve that subrequest on the same
isolate that is still busy awaiting it, which deadlocks until the proxy
times out. The failure was already documented on one handler's comment as
a known hazard; those three tools predated the finding and never got the
same treatment.

The fix (commit `2ac2f7d`): a shared `startVerdict` function now starts
verdicts in-process for both the HTTP route and every MCP submit path, and
`zyntry_get_verdict` reads the database directly. Zero self-fetches remain
in the MCP surface. Verified live: the same call that returned 522 came
back with the running verdict's row in 7 seconds.

If a verification gateway can deadlock on its own request routing, anything
can. The bug was found by exercising the product the way an agent would.

The same smoke run exposed a second design flaw: an evaluation tool call
that blocks until every verdict finishes loses its snapshot when the
client times out, even though the verdicts themselves complete on the
court's watchdog. Evaluations now persist their run list before and during
the batch, and the before/after aggregate is always computed live from the
verdicts table — nothing about a measurement depends on the client that
started it surviving. A `zyntry_list_verdicts` tool completes the
recovery path for interrupted submissions.

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
The harness is resume-safe; rows land as runs complete and this table
grows. Infra-errored runs (network outages, machine crashes) are excluded
and re-run, never counted.

| Case | Mode | Run | Result | Hidden suite | Calls |
|---|---|---|---|---|---|
| owasp-auth-bypass | baseline | 1 of 5 | `resolved-verified` | PASS | 25 |
| owasp-sql-injection | baseline | 1 of 5 | `resolved-verified` | PASS | 16 |
| owasp-sql-injection | baseline | 2 of 5 | `resolved-verified` | PASS | 18 |

Baseline mode = one agent, no courtroom pressure. A single glm-4.7 agent
clears the OWASP-derived sealed standards on these cases; the interesting
comparison (where the courtroom earns its fee) is the harder planted-flaw
and financial-correctness cases still in the queue.

<!-- BENCHMARK_RESULTS_CF -->


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
