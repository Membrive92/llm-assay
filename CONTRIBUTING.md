# Contributing

This is a single-maintainer project at present. The conventions below exist so that the repository reads the same whether one person or ten are working on it.

## Branches and commits

- `main` is always buildable. Work happens on short-lived branches named `<type>/<short-description>` (`feat/schema-oracle`, `docs/adr-0005`).
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`. Scope is the module name (`feat(oracles): add tool-trace subsequence match`).
- One logical change per commit. Dataset changes are separate commits from code changes.

## Change size and maintainer verification

Most code here is agent-written. The maintainer must be able to explain every part of the repository without an agent present; the limits below protect that.

- One module per change, under 200 changed lines of production code as a target and 400 as a hard limit (tests, datasets and docs excluded). Larger work is split into ordered, individually reviewed steps.
- One new abstraction per change, with its ADR or ARCHITECTURE entry in the same diff.
- Every change to a contract, module boundary, data flow or policy bumps the version header in `docs/ARCHITECTURE.md` and adds a row to its revision history.
- Every PR carries a review brief (what, why, how to verify by hand, what to read and in which order). The maintainer reads the brief and the docs *before* the diff.
- The maintainer merges. Agents open PRs and respond to review; they do not merge, do not push to `main`, and do not tick the checklist.
- A PR is not merged until the maintainer can restate, in their own words, what the change does and why it is shaped that way. If that requires asking the agent, the documentation is incomplete and the PR goes back.

## Pull requests

- Every PR runs the `pr` CI profile. A red canary blocks the merge without discussion.
- A change to an oracle's verdict logic must include or update fixture cases in `assay-oracles` with known verdicts.
- A change to a rubric bumps its version and re-runs calibration; the calibration result is attached to the PR.
- A change to a threshold or to the run policy defaults requires an ADR.

## Datasets

- Case IDs are immutable. A wrong case is fixed in place or retired with a `retired` tag; its ID is never reused.
- Knowledge-base documents must be licence-clean and small. No copyrighted material, no personal data.
- Every plane keeps exactly one canary. Adding a second requires a stated reason in the PR.
- `datasets/security/` holds exported corpus snapshots only. Payloads are not authored or edited in this repository; a new snapshot is a new file with a new version, never an edit of an existing one.

## Architecture decisions

Anything that a future reader would ask "why?" about gets an ADR in `docs/adr/`, following `0000-template.md`. ADRs are numbered sequentially and never renumbered. Superseding an ADR means adding a new one and updating the old one's status; the old text stays.

## Code

- Java 21. Records for data, sealed interfaces for closed sets, no Lombok, no reflection-based mapping in the core.
- Tests in `assay-core` and `assay-oracles` do not touch the network. Anything that needs a model runs against a fake `LlmClient` or is tagged `integration`.
- Formatting is enforced by the Maven build; do not hand-format.
