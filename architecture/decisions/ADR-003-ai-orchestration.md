# ADR-003: AI Orchestration — Deterministic Workflows with Single-Step LLM Calls

- **Status:** Proposed (awaiting approval)
- **Date:** 2026-10-30

## Context

The product needs some AI behavior, but the exact feature list is not yet confirmed (candidates: natural-language task capture, goal breakdown, prioritization, daily planning, a chat assistant over the user's tasks). Multi-agent designs add cost, latency, lossy context handoffs, and hard-to-reproduce failures.

## Open question (required before acceptance)

Which AI behaviors must the MVP include, and does any of them need an autonomous loop rather than a single call?

## Decision (provisional)

Use **one orchestrator, no multi-agent system, for the MVP.**

- **Deterministic code (no LLM):** CRUD, filtering, sorting, due-date and recurrence logic, reminders, permissions, validation, and applying approved changes in a DB transaction with an audit log entry.
- **LLM steps (each a single structured-output call):** free text to structured task, goal to subtasks, prioritization or summarization, and question answering over the user's tasks.
- Call the LLM provider **directly** through a thin internal client interface (so the provider is swappable and a fake client can be used in tests). Do not adopt LangChain.
- Tools are allow-listed (`create_task`, `update_task`, `search_tasks`). Write tools produce a **proposal**, never a direct write (see ADR-004).
- **Invalid output handling:** validate with Pydantic plus semantic checks (IDs exist and belong to the user, dates and enums valid). Retry at most twice with the validation error fed back, then fall back deterministically or return a clear error. Never partially write.
- Log raw output, validation errors, retry count, model and prompt version, and latency. Track the schema-failure rate and keep a regression eval set.
- Hosted vs local model (Ollama) is deferred. The client interface keeps both options open.

## Alternatives considered

- **Multi-agent system:** rejected for the MVP. It is justified only for trust-boundary isolation (for example, a component reading untrusted external content kept separate from the one with write access), real parallelism, or evals showing a single agent failing where a specialist succeeds.
- **LangGraph:** the preferred option if an agent loop is later needed, because its explicit state machine keeps control flow visible. Not needed for single-call steps.
- **LangChain:** rejected. Heavy abstraction and churn for little benefit over direct calls with structured output.

## Consequences

- Simple to trace, test, and evaluate; low cost and latency.
- Autonomous multi-step behavior is out of scope until needed and justified by evals.
- A thin client interface adds a small amount of code but keeps provider choice reversible.
