# ADR-0001: JVM stack instead of the Python evaluation ecosystem

**Status:** Accepted
**Date:** 2026-09-03

## Context

Evaluation tooling for LLM features is concentrated in Python: DeepEval, Ragas, promptfoo, Inspect, plus the notebook-centric workflows around them. Job postings for AI QA roles list Python and pytest almost without exception.

The author's production stack is Java (Playwright Java binding, Selenium, Rest-Assured, JUnit), and the target users of a harness like this are QA teams inside enterprise Java organisations that are adding LLM features to existing systems. Those teams will not adopt a second language and toolchain for one test category; they will either evaluate LLM features with what they have, or not evaluate them.

Spring AI provides the building blocks needed on the JVM: a provider-agnostic `ChatClient`, an `EmbeddingModel` abstraction, and an `Evaluator` interface with two LLM-as-judge implementations (`RelevancyEvaluator`, `FactCheckingEvaluator`). Community reports show those default evaluators produce inconsistent verdicts with their stock prompts, so they cannot be used as-is for acceptance.

## Decision

Build the harness in Java 21 on Maven, JUnit 5, Rest-Assured, Spring AI and Testcontainers. Use Spring AI's evaluators as a baseline reference only; define an independent `Oracle` contract and treat judge calibration as a deliverable.

The reasoning: the value of the harness is the evaluation method (oracles, run policy, calibration, canaries), and that method is language-independent. Implementing it on the JVM puts it where the target teams already are and where no comparable tool exists.

## Consequences

Favourable: direct reuse of the author's stack; integration with existing JVM CI pipelines; a niche with little competition.

Unfavourable: discoverability. Searches for "LLM evaluation" will surface Python tools first, and recruiters filtering on pytest/DeepEval will not find this repository. Mitigations: all datasets, rubrics, verdicts and results are plain JSON with documented schemas so Python tooling can consume them; the README leads with the method, not the language; the repository description and topics name the problem ("LLM feature evaluation", "AI QA") rather than the stack.

Unfavourable: fewer ready-made oracles. Semantic similarity, metamorphic testing and tool-trace comparison are implemented here rather than imported. This is more work and is also the part of the project that demonstrates judgment.

## Alternatives considered

**Python + pytest.** Best market fit and richest tooling. Rejected because it would duplicate what already exists, and because the author's differentiation is JVM test infrastructure, not Python.

**Python core with a Java adapter.** Rejected: two languages double the maintenance surface for a single-maintainer project, and the adapter would always lag the core.
