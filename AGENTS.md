# Agent instructions for llm-assay

Instructions for any coding agent working in this repository (Claude Code reads this through `CLAUDE.md`; Codex, Cursor and others read it directly). Keep this file short and operational. Explanations live in the documents it points to.

## What this is

A JVM evaluation harness for LLM-backed features. Read `README.md` for scope, `docs/ARCHITECTURE.md` for module contracts, and `docs/adr/` for the reasoning behind decisions you may be tempted to change. Do not re-derive a decision that has an ADR; if you disagree, propose a new ADR.

## Layout and boundaries

Maven multi-module. Dependency direction is strict and, from M0, enforced by the build:

```text
assay-e2e, assay-security  →  assay-sut-dify  →  assay-core  ←  assay-oracles
```

- `assay-core` imports nothing from this repository and nothing from Dify. The `Case`, `Verdict`, `Oracle`, `Observation` and `RunPolicy` types live here and are the contract everything else obeys.
- Dify-specific JSON shapes appear only in `assay-sut-dify`.
- Oracles never call the SUT. They receive an `Observation` and return a `Verdict`.
- The adversarial corpus (`datasets/security/corpus-<version>.json`) is external data authored by a separate CLI. `assay-security` reads and validates it; nothing in this repository creates, edits or extends payloads.

## Rules that do not bend

1. **Do not change acceptance semantics without an ADR.** Thresholds, default `n`, the confidence level, the Wilson choice and the INCONCLUSIVE rule are policy. A failing evaluation is fixed in the SUT configuration, the prompt, the dataset or the oracle, never by loosening the policy or raising `n` until it passes.
2. **Never fabricate evaluation output.** No hard-coded verdicts, no placeholder pass rates, no invented calibration figures (κ, recall) in docs or fixtures. If a number cannot be computed yet, write "not yet measured".
3. **Canaries must fail.** If a change makes a canary pass, the change is wrong or the canary is; find out which before continuing.
4. **Case IDs are immutable and never reused.** Retire with a `retired` tag.
5. **Judge and SUT are never the same model at the same version.**
6. **No secrets in code, fixtures, attachments or results.** Provider and Dify keys come from the environment (`.env.example` lists them). Anything written to Allure or `results/` passes through the secret scrub.
7. **Unit tests in `assay-core` and `assay-oracles` do not touch the network.** Use the fake `LlmClient`. Anything needing a model or Dify is tagged `integration`.
8. **Java 21, records for data, sealed interfaces for closed sets. No Lombok. No reflection-based mapping in core.**

## Maintainer comprehension comes first

Most of this code will be written by an agent. The project only has value if its single human maintainer can still explain every part of it without an agent present. Every rule in this section exists to keep that true, and it takes precedence over speed.

**Change size.** One task, one concern, one reviewable diff. Limits:

- A change touches one Maven module (plus its tests and the docs that describe it). Cross-module changes are split into ordered steps, each reviewed before the next starts.
- Target under 200 changed lines of production code per change; never over 400. Datasets, fixtures and docs do not count, but a large dataset change is still its own change.
- One new abstraction (interface, sealed hierarchy, module) per change, introduced with its ADR or ARCHITECTURE entry in the same diff.
- If the task cannot fit, stop and propose the split. Do not proceed with a large change because it was asked for in one sentence.

**Documentation moves with the code, in the same change.**

- A contract, module boundary, data flow or policy change updates the matching section of `docs/ARCHITECTURE.md` and bumps its version header (see the file). The bump is the signal to the maintainer that something structural moved.
- A decision with alternatives gets an ADR. A change that contradicts an existing ADR gets a superseding ADR, not a silent edit.
- Anything user-visible or behaviour-changing gets a line under *Unreleased* in `CHANGELOG.md`.
- Cosmetic or purely internal fixes do not touch the docs. Do not generate documentation churn to look thorough.

**The human verifies every change. The agent never merges.**

- Do not push to `main`, do not merge, do not resolve review threads, do not mark a checklist item as done on the maintainer's behalf.
- End every task with a **review brief** in the PR description, in this order: what changed (one paragraph); why (link to issue or ADR); how to verify it by hand, with the exact commands and what output to expect; what the maintainer should read to understand it, in reading order, with an estimate of the time it takes. Write the brief for someone who has not seen the diff.
- If a change introduces something the maintainer would need the agent to explain, the docs are incomplete. Fix the docs, not the explanation.

**Working style.**

- Prefer the smallest change that solves the stated problem. Do not refactor adjacent code as part of an unrelated task.
- When a requirement is ambiguous, state the ambiguity and the interpretation you chose in the review brief rather than picking silently.
- When a non-deterministic test fails, report the observed pass rate and interval, not just "failed". The interval is the diagnostic.

## Commands

Available from M0 onward. Until `pom.xml` exists, only the docs lint runs.

```text
mvn -B -ntp verify                       # full build, unit tests, no SUT
mvn -B -ntp -Ppr verify                  # PR profile: canaries at full n, rest at n=3, Testcontainers up
mvn -B -ntp -Pnightly verify             # everything at full n, calibration checks
mvn -B -ntp -pl assay-sut-dify verify -Dplane=grounding   # one plane
```

## Commits

Conventional Commits with the module as scope: `feat(oracles): ...`, `fix(sut-dify): ...`, `docs(adr): ...`. Dataset changes are separate commits from code changes.
