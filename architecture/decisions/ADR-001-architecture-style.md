# ADR-001: Architecture Style — Modular Monolith

- **Status:** Proposed (awaiting approval)
- **Date:** 2026-10-30

## Context

TaskFlow AI is a task-management product with an AI layer. It is being built from a blank repository, likely by one person or a small team. Task data is relational (users, projects, tasks, AI proposals, audit log), and several operations must be atomic (for example, applying an approved AI proposal and writing the audit log).

## Decision

Build a **modular monolith**: one deployable backend with strict internal module boundaries.

- Domain modules such as `tasks`, `projects`, `auth`, `ai_orchestration`, `notifications`.
- Each module owns its models and repositories and exposes a small public service interface. Modules do not read each other's tables or import each other's internals.
- Boundaries are enforced in CI (for example with `import-linter`), not by convention alone.
- `ai_orchestration` sits behind a narrow interface (for example `propose_actions(context) -> Proposal`) so it can be extracted later.
- Long-running AI work moves to a background worker process **from the same codebase** before any service split is considered.

## Alternatives considered

- **Microservices:** rejected for the MVP. No independent teams, no divergent scaling needs, and they add network failure modes, distributed transactions, tracing, and per-service CI/CD for no current benefit.
- **Unstructured monolith:** rejected. Without enforced boundaries it degrades into a tightly coupled codebase and makes later extraction costly.

## Consequences

- Simple ACID transactions across modules; one deploy, one test suite, fast iteration.
- A single bad deploy affects everything, which is acceptable at MVP scale.
- Boundary discipline requires CI enforcement and ongoing review.

## Revisit when

- AI workloads starve the API even with a separate worker process.
- A model must run on separate hardware.
- Multiple teams own different modules.
- Compliance or isolation requirements force separation.
