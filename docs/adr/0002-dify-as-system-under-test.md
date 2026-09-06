# ADR-0002: Dify as the system under test

**Status:** Accepted
**Date:** 2026-09-03

## Context

The harness needs a real system with LLM-backed features to evaluate. Three options were on the table: build a minimal application, evaluate an existing open-source application, or evaluate an isolated prompt-plus-model component with no application around it.

The requirement is coverage of three feature types under one deployment: retrieval-augmented answering (grounding failures), structured output generation (schema failures) and tool-using agents (trace and injection failures). The SUT must be self-hostable, driveable through an HTTP API, and able to run local models so that evaluation runs do not depend on paid APIs.

## Decision

Use Dify, self-hosted via Docker Compose, as the system under test. Dify provides a RAG pipeline, workflow apps with structured output, and agents with tools; every app type has a corresponding API; it runs against Ollama; and since v1.6 it has native two-way MCP support (consuming MCP servers as tools, and exposing apps as MCP servers). That last capability is what makes the security suite viable, because the adversarial corpus targets MCP and agent tool surfaces.

The Dify-specific code is confined to `assay-sut-dify`. The case and verdict models do not reference Dify.

## Consequences

Favourable: a credible SUT with no application code to write or maintain; black-box evaluation through an API, which mirrors how a QA team would work against a product they do not own; three planes covered by one environment.

Unfavourable: dependency on Dify's release cadence. API shape changes break the adapter. Mitigation: pin the Dify image version in the Testcontainers environment and upgrade deliberately, with the adapter's contract tests as the gate.

Unfavourable: Dify's licence is source-available with conditions (not plain Apache-2.0). Irrelevant for an external harness that consumes its API, but it must be stated in the README to avoid giving the impression that the harness redistributes Dify.

Unfavourable: less control over the internals than a self-built application. Retrieved chunks and tool traces are read back through whatever Dify exposes; where the API does not expose an intermediate artefact, the oracle for that artefact is limited to what can be inferred from the final output. This limitation is documented per oracle.

## Alternatives considered

**Self-built minimal application (RAG + structured output).** Full control and the option to inject regressions at will. Rejected because it doubles the work, and because a self-built SUT reads as a toy to a reviewer.

**Open WebUI.** Simpler than Dify, RAG plus tools, OpenAI-compatible API. Rejected because agent and workflow surfaces are thinner, which weakens the Agent plane.

**Onyx (formerly Danswer).** Enterprise RAG with connectors. Rejected for the same reason: strong on grounding, weak on structured output and agents.

**Isolated prompt-plus-model component.** Fastest to start. Rejected because it demonstrates nothing about integration, and integration is what the target roles ask for.
