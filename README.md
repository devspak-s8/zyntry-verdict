# Zyntry Verdict

An AI courtroom that verifies what the machines make.

Code. Infrastructure. Analysis. Any AI-generated artifact with a testable
standard: three adversarial agents investigate it, a Challenger writes new
tests your own checks don't have, a failing test is the only thing that
settles a dispute, and the standard is sealed (the agents being judged
cannot read or run the tests that judge them).

Verified domains, on the live gateway today:

- Code: planted security flaws, OWASP-derived standards
- Infrastructure: AI-generated cloud configs vs CIS-derived policy
- Analysis: AI-generated statistics vs sealed standards of statistical truth

Try it:

    npx zyntry-verdict@latest signup
    npx zyntry-verdict@latest init     # scan YOUR repo
    npx zyntry-verdict@latest run      # three agents review it, live

Live gateway: https://verdict.zyntry.space
Full docs: https://zyntry.space/products/zyntry-verdict/docs

## Public records

- [Results and methodology](RESULTS.md): benchmarks, measured token costs,
  the protocol robustness bug we found and fixed, and the losses we publish.
- [Dogfood](DOGFOOD.md): we run the engine through its own courtroom, with
  our own test suite as the sealed standard.
- [Security audit](SECURITY_AUDIT.md): 20/20 live pentest, secrets scan,
  sandbox review, and every gap that was fixed.

The engine itself is private. These are the records we publish.

## The category

Cisco and OpenAI now ship AI agents that write kernel-level defenses. AI
writes code, configs, analysis, and soon policy and contracts. Every one of
those artifacts makes claims about itself. Nobody verifies the claims.

Verification of what the machines make is the category. Code is merely the
first, most mechanically verifiable province of it. The courtroom is
domain-agnostic by construction: it never spoke code in the first place.
It speaks actions, evidence, claims, and verdicts.
