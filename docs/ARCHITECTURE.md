# Architecture

| | |
| --- | --- |
| **Document version** | 0.1.0 |
| **Applies to code** | pre-M0 (no code yet) |
| **Last structural change** | 2026-09-06, initial contracts |

The version follows semver on the *architecture*, not on the code: patch for clarifications, minor for a new module, oracle type or policy parameter, major for a change to `Case`, `Verdict`, `Oracle` or the dependency direction. Every bump is listed in the revision table at the end of this document and in `CHANGELOG.md`. A code change that alters anything this document describes bumps the version in the same commit.

This document describes the module contracts, the data flow of an evaluation run, and the three design areas that carry most of the risk: non-determinism, judge calibration and the security corpus contract. It is the reference for anyone reading the code; the *why* behind each choice lives in [`adr/`](adr/).

## 1. Modules and dependency direction

```text
assay-e2e ──────┐
assay-security ─┼──► assay-sut-dify ──► assay-core ◄── assay-oracles
                └─────────────────────────────┘
```

`assay-core` depends on nothing inside the repository. `assay-oracles` depends only on core. Everything that knows about Dify lives in `assay-sut-dify`. The security and E2E modules sit on top and may use any of the three below them. No module depends on a sibling at the same level.

### assay-core

Owns the three contracts the rest of the system is built on.

```java
public record Case(
    String id,                 // stable, e.g. "grounding-0042"
    Plane plane,               // GROUNDING | STRUCTURED | AGENT
    Input input,               // user message, attachments, conversation state
    Expectation expectation,   // golden answer, schema ref, expected tool trace, metamorphic relation
    Set<String> tags,          // free-form: "canary", "owasp-asi-01", "citation-required"
    List<OracleRef> oracles    // which oracles decide this case, and how they combine
) {}

public sealed interface Verdict permits Pass, Fail, Inconclusive {
    double score();            // [0,1]; 1.0 for hard pass, 0.0 for hard fail
    String rationale();        // human-readable, one paragraph max
    Evidence evidence();       // retrieved chunks, judge output, schema errors, tool trace
}

public interface Oracle {
    OracleId id();
    Verdict judge(Case c, Observation o);
}
```

An `Observation` is what the SUT adapter returns for one execution: raw output, latency, token usage where available, retrieved context, and the tool-call trace for agents. Oracles never call the SUT; they only look at observations.

The module also holds the `RunPolicy` (section 3), the `LlmClient` abstraction (a thin wrapper over Spring AI's `ChatClient` and `EmbeddingModel`, so that oracles can be tested with a fake), and the result aggregation that feeds the report.

### assay-oracles

| Oracle | Plane(s) | Decides | Deterministic |
| --- | --- | --- | --- |
| `SchemaOracle` | Structured, Agent | Output conforms to a JSON Schema; no extra properties when the schema is closed | Yes |
| `SemanticSimilarityOracle` | Grounding | Cosine similarity between embedding of output and golden answer exceeds a per-case threshold | Yes given fixed embedding model |
| `FactCheckOracle` | Grounding | Every claim in the output is supported by the retrieved context (LLM-as-judge, small dedicated model) | No |
| `RubricJudgeOracle` | All | Output scored against a versioned rubric by a judge model; verdict from score threshold | No |
| `MetamorphicOracle` | Structured, Grounding | A declared relation holds between two observations (paraphrased input yields structurally equal output; reordered constraints yield same result) | Depends on inner oracle |
| `ToolTraceOracle` | Agent | Sequence of tool calls matches the expected pattern (exact, subsequence, or forbidden-set) | Yes |

A seventh oracle, `RefusalOracle` (did the agent refuse, comply or partially comply with an adversarial instruction), is security-specific and lives in `assay-security` rather than here, because its rubric is tied to the corpus taxonomy.

Non-deterministic oracles are themselves under test: `assay-oracles` ships a fixture set with known verdicts and a test that fails if judge agreement drops below the calibrated floor (section 4).

### assay-sut-dify

Rest-Assured client for the Dify API: chat messages, workflow runs, conversation retrieval, and the app-level endpoints needed to read back retrieved chunks and tool traces. A `DifyEnvironment` class wraps Testcontainers to start the Dify compose stack and an Ollama container, load the fixed knowledge base, and expose base URLs and API keys to the tests.

The adapter maps Dify responses to `Observation`. It is the only place where Dify's JSON shape appears.

### assay-security

Reads an adversarial payload corpus supplied as JSON by an external CLI (section 5), turns each payload into a `Case` on the Agent plane, executes it through the Dify adapter and decides it with `ToolTraceOracle` plus a `RefusalOracle` (did the agent refuse, comply, or partially comply). Scores each finding with AIVSS and tags it with its OWASP ASI / MCP Top 10 class. Output uses the same `Verdict` shape as the functional suite.

### assay-e2e

Playwright for Java. Scoped to cases where the browser changes what the user sees relative to the API response: streaming truncation, citation rendering, markdown sanitisation. It reuses cases from `datasets/` and adds a UI observation to them; it does not define its own cases.

## 2. Data flow of one evaluation run

```text
datasets/*.json ──► CaseLoader ──► RunPolicy ──► SutAdapter (×n) ──► Observation[]
                                                                        │
                                            Verdict[] ◄── Oracle(s) ◄───┘
                                                │
                                Aggregator (Wilson interval, threshold per plane)
                                                │
                                   Allure report + JSON results file
```

Each case executes `n` times (default per plane, overridable per case). Every execution yields an observation; every observation yields one verdict per oracle attached to the case. The aggregator combines per-oracle verdicts into a per-execution verdict (all-pass, or weighted where the case declares weights), then computes the pass rate across the `n` executions and its confidence interval.

Results are written twice: to Allure for humans, and to `target/results/<suite>-<timestamp>.json` for regression comparison between runs.

## 3. Non-determinism policy

A single execution of an LLM feature is a sample, not a measurement. The harness therefore:

1. Executes every case `n` times at the SUT's production temperature. Default `n = 5` for Grounding and Structured, `n = 10` for Agent and for any case tagged `canary`.
2. Computes the pass rate and a Wilson score interval at 95 % confidence. Wilson is chosen over the normal approximation because it behaves correctly at small `n` and at pass rates near 0 or 1, which is exactly where these tests live.
3. Compares the **lower bound** of the interval to the plane's threshold. Default thresholds: Grounding 0.90, Structured 0.98, Agent 0.95. A case passes when its lower bound clears the threshold.
4. Marks a case `INCONCLUSIVE` when the interval straddles the threshold and `n` is below the maximum; CI treats inconclusive as failure unless the case is tagged `flaky-allowed` with a linked issue.

Temperature 0 is not used as a default. It narrows the sample to one mode of the output distribution and hides failures that occur in production. Reference runs at temperature 0 are available as a separate profile for debugging, never for acceptance.

Cost control: `n` is configurable per suite through Maven properties, and the CI profile runs the canary subset at full `n` and the rest at `n = 3` on pull requests, full `n` nightly.

## 4. Judge calibration

Every LLM-as-judge oracle is a classifier, and its accuracy has to be measured before its verdicts mean anything. The procedure:

1. A calibration set of at least 100 observations per rubric, labelled by a human with the same rubric. Labels stored in `datasets/calibration/`.
2. The judge runs over the calibration set; agreement is reported as Cohen's κ and as precision/recall on `FAIL`, because a judge that never flags failures is the dangerous case.
3. A rubric is accepted for use when κ ≥ 0.70 and recall on `FAIL` ≥ 0.85. Below that, the rubric or the judge model changes, not the threshold.
4. Rubrics are versioned (`rubric-id@version`) and the version is recorded in every verdict. A rubric change re-runs calibration.
5. `assay-oracles` includes a test that re-runs a fixed subset of the calibration set and fails the build when agreement drops below the accepted floor, which catches silent judge-model drift (provider updates, Ollama model re-pulls).

Judge model and SUT model are never the same model at the same version.

## 5. Security corpus contract

The adversarial corpus is produced outside this repository by a Python CLI and consumed here as data. The contract is a JSON file per corpus version:

```json
{
  "corpus_version": "1.2.0",
  "taxonomy": "owasp-asi-2026",
  "payloads": [
    {
      "id": "asi-01-0007",
      "class": "ASI-01",
      "mcp_top10": ["MCP-03"],
      "vector": "tool_output",
      "payload": "...",
      "expected_behaviour": "refuse",
      "forbidden_tools": ["send_email", "http_request"],
      "aivss_base": 7.1
    }
  ]
}
```

The exported corpus snapshot is committed under `datasets/security/corpus-<version>.json` so that a run is reproducible from the repository alone; the CLI remains where payloads are authored. `assay-security` validates the file against a schema on load and rejects unknown taxonomy versions. The harness does not generate or mutate payloads; that responsibility stays with the CLI, which keeps one source of truth for the taxonomy and the scoring.

## 6. Datasets

```text
datasets/
├── grounding/
│   ├── knowledge-base/        Fixed documents loaded into Dify (versioned, small, licence-clean)
│   ├── cases.json
│   └── canary.json            One case with a deliberate regression injected through a prompt variant
├── structured/
│   ├── schemas/
│   ├── cases.json
│   └── canary.json
├── agent/
│   ├── cases.json
│   └── canary.json
├── security/
│   └── corpus-<version>.json   Exported adversarial corpus (see section 5); read-only in this repository
└── calibration/
    └── <rubric-id>@<version>.json
```

Every plane has a canary: a case whose SUT configuration includes a known defect (a prompt that drops the citation instruction, a schema with a loosened field). The canary must fail. A green canary means the harness is not detecting what it claims to detect, and the build fails.

## 7. Reporting

Allure is the human-facing report. Each failed execution attaches: the exact request sent to the SUT, model and parameters, raw output, retrieved context, tool trace, each oracle's verdict with rationale and evidence, and the rubric version. A failure without enough attached context to reproduce it by hand is a harness bug.

Everything written to Allure or to the results file passes through a secret scrub that redacts values matching known provider-key formats and any value loaded from a `*_API_KEY` or `DIFY_API_KEY_*` variable. A scrub failure fails the build. The scrub ships with M1, the first milestone that writes results.

The JSON results file is the machine-facing output. A `compare` goal diffs two results files and lists cases whose pass-rate interval moved across the threshold, which is how prompt and model changes are gated.

## 8. CI

Two profiles. `pr` starts the Testcontainers environment, runs canaries at full `n` and the rest at reduced `n`, and fails on any threshold breach or inconclusive verdict. `nightly` runs everything at full `n`, re-runs judge calibration checks, publishes Allure, and archives the results JSON for trend comparison.

## 9. Conventions

- Java 21, records for data, sealed interfaces for closed sets. No Lombok.
- One assertion concept per test method; the assertion is on the aggregated verdict, never on a single execution.
- Datasets are code: changes go through review, and a case ID is never reused.
- Every ADR follows the template in `adr/0000-template.md`.

## 10. Revision history

| Version | Date | Change | Driven by |
| --- | --- | --- | --- |
| 0.1.0 | 2026-09-06 | Initial contracts, module layout, non-determinism policy, calibration procedure, corpus contract | ADR-0001 to ADR-0004 |
