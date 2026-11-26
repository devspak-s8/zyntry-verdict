# Zyntry Verdict

An AI courtroom that verifies code against sealed hidden standards. Three
adversarial agents (investigator, proponent, challenger) examine AI-written
code. Every claim needs a failing test to prove it. The standard is sealed:
agents can neither read nor run the tests that judge them.

Try it on a demo case:

    npx zyntry-verdict@latest signup
    npx zyntry-verdict@latest run multi-tenant-auth

Or point the courtroom at YOUR repo:

    cd your-repo
    npx zyntry-verdict@latest init     # scans the repo, generates .zyntry.case.json
    npx zyntry-verdict@latest run      # submits your code; the court reviews it live

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

