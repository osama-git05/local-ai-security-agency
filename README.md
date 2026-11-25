# local-ai-security-agency
A local-first, multi-agent system that reviews lab evidence, verifies security findings, and generates human-approved reports.

## Repository Guide

This repository is being built in six increments from `BUILD_BLUEPRINT.md`.

- `config/` - local scope and runtime configuration
- `src/agency/` - contracts, storage, model adapter, roles, workflow, reporting, and API
- `fixtures/` - synthetic evidence and evaluation cases
- `tests/` - level-by-level acceptance tests
- `scripts/` - smoke, benchmark, and demonstration commands
- `frontend/` - local operator dashboard
- `docs/` - architecture and evaluation documentation
- `var/` - gitignored local databases, evidence objects, exports, and gate results

Use the purpose-named Markdown guides in each directory for implementation boundaries and acceptance work.
