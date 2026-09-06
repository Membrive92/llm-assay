# Security policy

## Scope

`llm-assay` evaluates the behaviour of LLM-backed features and ships an adversarial test suite. Two things follow from that.

The repository contains, by design, prompt-injection payloads and other adversarial inputs under `datasets/agent/` and `datasets/security/`. They are test data. They are not exploits against Dify, against any model provider, or against any third party, and they must not be used outside a system you own or are authorised to test.

The harness itself is a test tool that talks to a system under test through its API. It does not run as a service and does not expose endpoints.

## Reporting a vulnerability

If you find a problem in the harness that could cause it to leak credentials (for example, a provider key written into an Allure attachment or a results file), to execute a payload against an unintended target, or to misreport a security finding as a pass, report it privately through GitHub's security advisory form for this repository rather than as a public issue. Expect an acknowledgement within seven days.

Findings about the models or about Dify belong to their respective maintainers.

## Handling of secrets

Provider keys and Dify app keys are read from environment variables (see `.env.example`) and are never committed. From milestone M1 (the first that writes results), Allure attachments and results files pass through a secret scrub before they are written, and a scrub failure fails the build; the mechanism is described in `docs/ARCHITECTURE.md`, section 7. If you find a key in a committed artefact, rotate it first and report second.
