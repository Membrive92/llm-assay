# llm-assay

**Statistical evaluation harness for LLM-backed features, on the JVM.**

An *assay* determines the purity of a metal from a sample and reports a figure with its uncertainty. This harness does the same for an application feature backed by a language model: it samples repeated executions, decides each one with an explicit oracle, and reports a pass rate with a confidence interval. One execution is a sample, not a measurement.

It answers two questions with one case model, one verdict model and one report:

- **Functional:** does the feature do what it should, given that the same input can produce different outputs on every run?
- **Security:** does the feature do what it should not, when the input is adversarial?

**Status:** design phase. No runnable code yet. See [Roadmap](#roadmap).

---

## Why this exists

Classical test automation rests on three assumptions: identical input yields identical output, an oracle exists that decides correctness, and each run produces one pass/fail. An LLM-backed feature violates all three. Output varies with temperature and sampling; "correct" is often a judgment rather than a comparison; and a single run tells you almost nothing about the failure rate.

Most tooling that addresses this is Python-first (DeepEval, Ragas, promptfoo, Inspect). That leaves a gap for the systems where LLM features are actually being added in 2026: Java back-ends in energy, banking, insurance and industrial software, maintained by QA teams that already run JUnit, Rest-Assured and Playwright. This harness targets that gap. It is not a port of a Python framework; it is an evaluation layer designed around JVM test infrastructure and CI conventions.

## What it tests

The system under test (SUT) is [Dify](https://github.com/langgenius/dify), self-hosted through Docker Compose. One deployment exposes three kinds of LLM feature behind a documented HTTP API, which is enough surface to exercise every oracle type without building a demo application.

| Plane | Dify feature | Dominant failure mode | Primary oracle |
| --- | --- | --- | --- |
| Grounding | Chat app over a knowledge base (RAG) | Hallucination, unsupported claims, wrong retrieval | Fact-check against retrieved chunks; semantic similarity to golden answer |
| Structured output | Workflow, natural language to JSON | Schema violations, invented fields, wrong types | JSON Schema validation; metamorphic relations (paraphrase preserves structure) |
| Tool-using agent | Agent with tools and MCP servers | Wrong tool calls, prompt injection through tool output, data exfiltration | Tool-trace comparison; adversarial corpus classified by OWASP ASI / MCP Top 10 |

The first two planes form the functional suite. The third is shared between the functional and security suites.

The choice of SUT is a design decision, not a dependency: the SUT adapter is one module, and the case and verdict models know nothing about Dify. See [ADR-0002](docs/adr/0002-dify-as-system-under-test.md).

## Technology

| Concern | Choice | Notes |
| --- | --- | --- |
| Language / build | Java 21, Maven multi-module | LTS; records and sealed types for the case/verdict model |
| Test runner | JUnit 5 | Parameterized tests over datasets; tags per oracle type and plane |
| HTTP client for the SUT | Rest-Assured | Dify API adapter |
| LLM access | Spring AI (`ChatClient`, `EmbeddingModel`, `Evaluator`) | Provider-agnostic; Ollama locally, hosted APIs for reference runs |
| Local models | Ollama | Generation, embeddings and a small fact-check model (e.g. MiniCheck) for the judge |
| Environment | Testcontainers | Dify compose stack plus Ollama, reproducible in CI |
| Schema validation | JSON Schema (networknt validator) | Structured-output oracle |
| UI-level checks | Playwright for Java | Only where the UI transforms the model output (streaming, citations, markdown) |
| Reporting | Allure | Full trace per failed case: prompt, parameters, raw output, verdict, evidence |
| Security corpus | JSON contract with an external adversarial payload CLI | Payloads tagged by ASI / MCP Top 10 class; AIVSS scoring |

Spring AI ships two `Evaluator` implementations (`RelevancyEvaluator`, `FactCheckingEvaluator`), both LLM-as-judge. They are used as a baseline only. Their default prompts produce inconsistent verdicts on anything beyond trivial cases, so the harness defines its own `Oracle` contract and treats judge calibration as a first-class deliverable. See [ADR-0001](docs/adr/0001-jvm-stack.md).

## Repository layout

```text
llm-assay/
├── .github/               CI workflow, PR and issue templates, Dependabot
├── assay-core/            Case, Verdict, Oracle contracts; run policy; statistics; provider-agnostic LLM client
├── assay-oracles/         Schema, semantic similarity, LLM-as-judge, metamorphic, tool-trace oracles
├── assay-sut-dify/        Rest-Assured adapter for the Dify API; Testcontainers environment
├── assay-security/        Adversarial corpus reader, payload execution, AIVSS / ASI scoring
├── assay-e2e/             Playwright checks on the chat UI
├── datasets/              Versioned golden sets and one canary per plane; committed adversarial corpus snapshot
├── docs/
│   ├── ARCHITECTURE.md    Module contracts, data flow, statistics, calibration, reporting
│   └── adr/               Architecture decision records
├── .env.example           Provider, SUT and run-policy variables; copy to .env (git-ignored)
├── .editorconfig / .gitattributes / .sdkmanrc / .markdownlint.yml
├── .gitignore
├── AGENTS.md              Instructions for coding agents; CLAUDE.md imports it
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE                Apache-2.0
├── SECURITY.md            Scope of the adversarial datasets; how to report harness vulnerabilities
└── README.md
```

## Core concepts

A **Case** is an input, an expectation, metadata (plane, tags, risk class) and a reference to the oracle set that decides it. Cases live in `datasets/` as JSON and are versioned with the code.

An **Oracle** takes a case and an observed output and returns a **Verdict**: `PASS`, `FAIL` or `INCONCLUSIVE`, with a score in `[0, 1]`, a rationale, and the evidence it used (retrieved chunks, judge response, schema errors, tool trace). Oracles compose: a case can require a schema oracle and a judge oracle to both pass.

A **Run policy** decides how many times a case executes and how the verdicts aggregate. The default is `n` runs per case with a Wilson score interval on the pass rate and a threshold per plane. A single green run is not a pass.

A **Suite** groups cases by plane and purpose. `functional` and `security` are the two shipped suites. Both produce the same verdict shape and land in the same report.

## Getting started

Not yet available. When the first milestone lands, this section will cover: Docker prerequisites, pulling the Ollama models, starting the Dify stack through Testcontainers, and running a single plane with `mvn -pl assay-sut-dify verify -Dplane=grounding`.

## Roadmap

- [ ] **M0. Contracts.** `assay-core`: Case, Verdict, Oracle interfaces; run policy with Wilson interval; provider-agnostic client. Unit-tested against fakes, no SUT.
- [ ] **M1. Grounding plane.** Dify adapter; RAG chat app over a fixed document set; schema, similarity and fact-check oracles; first golden set with a deliberate canary regression; secret scrub on every written artefact.
- [ ] **M2. Judge calibration.** Human-labelled sample; agreement metrics; rubric versioning. Documented in `docs/`.
- [ ] **M3. Structured-output plane.** Workflow adapter; JSON Schema oracle; metamorphic relations.
- [ ] **M4. Agent plane and security suite.** Tool-trace oracle; corpus contract; AIVSS scoring; shared report.
- [ ] **M5. CI.** Testcontainers pipeline; Allure publishing; threshold gates.
- [ ] **M6. E2E subset.** Playwright checks where the UI alters model output.

## Design decisions

All non-trivial choices are recorded as ADRs in [`docs/adr/`](docs/adr/). The ones that shape everything else:

- [ADR-0001](docs/adr/0001-jvm-stack.md): JVM stack instead of the Python evaluation ecosystem
- [ADR-0002](docs/adr/0002-dify-as-system-under-test.md): Dify as the system under test
- [ADR-0003](docs/adr/0003-single-harness-functional-and-security.md): functional and security suites in one harness
- [ADR-0004](docs/adr/0004-non-determinism-policy.md): repeated execution and statistical acceptance

## Reading guide

If you are evaluating this repository rather than using it, this order gives the most information per minute:

1. This README, for scope and status.
2. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) sections 3 and 4, for how non-determinism and judge calibration are handled. These two sections are the substance of the project.
3. [ADR-0004](docs/adr/0004-non-determinism-policy.md), for why a single execution is never a pass.
4. [`AGENTS.md`](AGENTS.md), for how agent-written code is kept reviewable by one maintainer.

## Non-goals

- Benchmarking foundation models. The unit under test is an application feature, not a model.
- Replacing Python evaluation tooling. Datasets, rubrics and verdicts are plain JSON so that either side can consume the other's output.
- Security-scanning Dify itself. The security suite targets the behaviour of the LLM feature, not the platform's code.

## License

[Apache-2.0](LICENSE).
