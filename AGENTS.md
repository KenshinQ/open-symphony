# Symphony TypeScript

This directory contains the TypeScript agent orchestration service that polls Linear, creates per-issue workspaces, and runs Codex in app-server mode.

## Environment


## Codebase-Specific Conventions

- Runtime config is loaded from `WORKFLOW.md`
- Keep the implementation aligned with [`SPEC.md`](SPEC.md) where practical.
  - The implementation may be a superset of the spec.
  - The implementation must not conflict with the spec.
  - If implementation changes meaningfully alter the intended behavior, update the spec in the same
    change where practical so the spec stays current.
- Workspace safety is critical:
  - Never run Codex turn cwd in source repo.
  - Workspaces must stay under configured workspace root.
- Orchestrator behavior is stateful and concurrency-sensitive; preserve retry, reconciliation, and cleanup semantics.
- Follow `docs/logging.md` for logging conventions and required issue/session context fields.

## Tests and Validation

Run targeted tests while iterating, then run full gates before handoff.

## Required Rules

- Keep changes narrowly scoped; avoid unrelated refactors.


Validation command:


## PR Requirements

- PR body must follow `.github/pull_request_template.md` exactly.
- Validate PR body locally when needed:

## Docs Update Policy

If behavior/config changes, update docs in the same PR:

- `README.md` for implementation and run instructions.
- `WORKFLOW.md` for workflow/config contract changes.
