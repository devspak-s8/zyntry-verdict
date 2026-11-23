# Zyntry Verdict

An AI courtroom that verifies code against sealed hidden standards. Three
adversarial agents (investigator, proponent, challenger) examine AI-written
code. Every claim needs a failing test to prove it. The standard is sealed:
agents can neither read nor run the tests that judge them.

Try it:

    npx zyntry-verdict signup
    npx zyntry-verdict run multi-tenant-auth

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
