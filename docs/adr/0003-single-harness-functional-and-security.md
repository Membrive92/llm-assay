# ADR-0003: Functional and security suites in one harness

**Status:** Accepted
**Date:** 2026-09-03

## Context

The author maintains a separate Python CLI for adversarial testing of agents and MCP clients: a payload corpus tagged by OWASP ASI and MCP Top 10 class, a transport interceptor, and unified AIVSS / ASI / MITRE ATLAS scoring. The question was whether this harness should cover functional evaluation only, with a documented hand-off to that CLI, or absorb security testing as a second suite.

The argument for separation is narrative clarity: a repository that does one thing is easier to explain. The argument for integration is that the two questions ("does it do what it should" and "does it do what it should not") are decided on the same observations, with the same run policy, and belong in the same report when a prompt or model changes.

## Decision

One harness, two suites. `functional` and `security` share the `Case` and `Verdict` models, the run policy, the SUT adapter and the report. The security suite does not generate payloads; it consumes a versioned JSON corpus exported by the external CLI, validated against a schema on load. The CLI remains the single source of truth for the taxonomy and the AIVSS scoring.

## Consequences

Favourable: one report answers both questions for a given SUT version, which is what a release gate needs. The corpus contract keeps the CLI and the harness decoupled; either can evolve without the other, as long as the contract version is respected.

Unfavourable: scope. Two suites are more surface to build and to explain. Mitigation: the README states the two questions in its first lines and the module layout makes the split visible; the Agent plane, shared by both suites, is built once.

Unfavourable: corpus drift. A taxonomy change in the CLI that is not reflected in the contract schema breaks the security suite at load time. This is intentional: failing loudly on an unknown taxonomy version is preferable to silently misclassifying findings.

## Alternatives considered

**Functional only, with a documented integration point.** Cleaner story. Rejected because the integration point would be theoretical until someone built it, and because release gating on LLM features needs both answers in one place.

**Re-implement the adversarial corpus in Java.** Rejected: duplicates a maintained asset and creates two taxonomies that will diverge.
