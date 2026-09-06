# ADR-0004: Repeated execution with statistical acceptance

**Status:** Accepted
**Date:** 2026-09-03

## Context

LLM outputs vary between executions of the same input at any temperature above zero, and production features run above zero. A test that executes a case once and asserts on that execution measures nothing: it passes or fails by sampling luck, and a green build carries no information about the failure rate users will see.

Two common workarounds were considered and rejected as defaults. Setting temperature to zero collapses the output distribution to one mode and hides the failures that occur in production. Retrying until pass turns every flaky failure into a pass and is indistinguishable from not testing.

## Decision

Every case executes `n` times at the SUT's production sampling parameters. The pass rate across executions is reported with a Wilson score interval at 95 % confidence, and the case passes when the interval's lower bound clears a threshold set per plane. Defaults: `n = 5` (Grounding, Structured), `n = 10` (Agent, and any case tagged `canary`); thresholds 0.90 / 0.98 / 0.95. A case whose interval straddles the threshold is `INCONCLUSIVE`; CI treats inconclusive as failure unless the case carries a `flaky-allowed` tag with a linked issue, which is a documented exception, not a default.

Wilson is used rather than the normal approximation because it remains valid at small `n` and at pass rates near 0 or 1, which is where acceptance tests operate.

## Consequences

Favourable: a pass means something quantified. Regressions between prompt or model versions are detected as interval shifts, not as single-run coin flips. Canaries at `n = 10` give the harness a self-check that is itself statistically meaningful.

Unfavourable: cost and time scale linearly with `n`. Mitigation: `n` is a Maven property; the PR profile runs canaries at full `n` and everything else at `n = 3`, the nightly profile runs full `n`; local models through Ollama keep the marginal cost of an execution near zero.

Unfavourable: thresholds are policy, not science. The defaults above are starting points and must be revisited once real pass-rate distributions per plane are observed. Changing a threshold is an ADR-level decision because it changes what "pass" means.

## Alternatives considered

**Temperature 0, single execution.** Fast and deterministic on many providers. Rejected as a default for the reason above; kept as a debugging profile.

**Fixed `n` with simple majority.** Simpler to explain. Rejected because it gives the same answer for 3/5 and 300/500, and the harness needs to say how confident it is.

**Sequential testing (SPRT).** Fewer executions on clear-cut cases. Deferred: attractive for cost, but harder to explain in a report, and premature before real distributions are known.
