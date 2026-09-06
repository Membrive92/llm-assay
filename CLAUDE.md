@AGENTS.md

## Claude Code

- Use plan mode for any change under `assay-core/` or to `docs/adr/`; those are contracts.
- When asked to "make the evaluation pass", stop and ask which of SUT configuration, prompt, dataset or oracle is in scope. Policy is not.
- Before starting any task, estimate the diff size against the limits in AGENTS.md. If it does not fit, propose the split first and wait.
- Never open more than one PR per task without being asked.
