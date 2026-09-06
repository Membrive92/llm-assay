## Review brief

### What changed
<!-- One paragraph. Module, and what it now does that it did not before. -->

### Why
<!-- Link the issue or ADR. If this changes a threshold, run policy or oracle verdict logic, link the ADR that authorises it. -->

### How to verify by hand
<!-- Exact commands and the output to expect. Written for someone who has not seen the diff. -->

### What to read to understand it
<!-- Files in reading order, with an estimate of the time needed. -->

## Size

- [ ] One module (plus its tests and docs)
- [ ] Under 200 lines of production code changed (hard limit 400) — actual: ___
- [ ] At most one new abstraction, documented in the same diff

## Checklist

- [ ] Commits follow Conventional Commits with a module scope
- [ ] Oracle verdict logic changed → fixture cases added or updated in `assay-oracles`
- [ ] Rubric changed → version bumped and calibration result attached
- [ ] Threshold or run policy changed → ADR added or updated
- [ ] Dataset changed → case IDs immutable, no personal data, licence-clean sources
- [ ] Documentation updated where behaviour changed (README, ARCHITECTURE, ADR)
- [ ] `docs/ARCHITECTURE.md` version bumped and revision row added, if anything structural moved

## Maintainer sign-off (filled by the maintainer, not the agent)

- [ ] I read the brief and the docs before the diff
- [ ] I can explain what this change does and why it is shaped this way, without the agent

## Evidence

<!-- Paste the relevant Allure summary or results diff for evaluation-affecting changes. -->
