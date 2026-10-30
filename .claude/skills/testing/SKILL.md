---
name: testing
description: Testing practice across backend, frontend and AI code - test pyramid, what to test at which layer, determinism, fixtures, failure injection, flaky-test policy, and AI eval method. Use when writing or reviewing tests, adding a feature, fixing a bug, or changing a prompt or model.
---

# Testing

Reusable testing practice. CI wiring and thresholds for this project are in `plans/09-testing-and-evaluation.md`.

## Principles
- Test behavior and contracts, not implementation. A refactor should not break tests that still describe correct behavior.
- Every bug fix starts with a failing test that reproduces it.
- Tests are deterministic and offline: no real network, no real clock, no randomness without a fixed seed.
- A test should fail for one reason and say which. Name tests after the behavior: `test_completing_recurring_task_creates_next_occurrence`.
- Prefer fewer, meaningful tests over coverage numbers. Use coverage to find untested branches, not as a goal.

## Choosing the layer
| Layer | Use for | Avoid |
|---|---|---|
| Unit | Pure logic: state machines, parsing, validators, calculations | Mocking the DB or framework |
| Integration | Router → service → real database; constraints, transactions, migrations | Mocking your own modules |
| End-to-end | A few critical user journeys | Re-testing details covered below |
| Contract | Boundaries with external systems (LLM provider, OpenAPI) | Running in every CI job if slow or costly |

Push logic down into pure functions so most behavior is unit-testable. Test the database with the **same engine as production**.

## Fixtures and data
- Build data with factories, not large shared fixtures. Each test creates only what it needs.
- Isolate tests with a transaction rollback or a clean schema per test; tests must pass in any order and in parallel.
- Inject time, ID generation and randomness as dependencies so tests control them.
- Do not use production data; use synthetic data with realistic edge shapes (unicode, long text, empty values, boundary dates, DST).

## What every feature needs
1. Success path.
2. Validation failures (each invalid-input class).
3. Authorization: **cross-user access is denied** for every resource and action.
4. Concurrency or idempotency where operations can repeat or race.
5. Failure injection for multi-step writes: force an error mid-operation and assert nothing partial persisted.
6. Boundary values: empty, one, max, off-by-one, time-zone edges.

Use property-based tests (Hypothesis) for rules with many input combinations, such as recurrence or transition tables.

## Test doubles
- Fake external dependencies at the narrowest boundary you own (for example an `LLMClient` interface), not by patching library internals.
- Prefer fakes (working simplified implementations) over mocks that assert call sequences.
- A fake must be able to simulate failure modes: timeout, malformed output, rate limit.

## Frontend tests
- Query by role and visible text, as a user would. Avoid test IDs unless there is no accessible alternative.
- Mock at the network layer (MSW). Assert what the user sees after success and after failure.
- Keep end-to-end tests few and stable; never let them become the main test suite.

## Testing AI behavior
LLM output is probabilistic, so test in two tiers:
1. **Deterministic tests (CI):** use a fake LLM client to drive the orchestration code - valid output, invalid JSON, wrong schema, semantic violations, timeouts - and assert retries, fallbacks and that nothing is written without approval.
2. **Evals (on demand and before prompt/model changes):** run a versioned dataset against the real model and report metrics.

Eval method:
- Dataset of `{input, context, expected}` cases; start small (about 50), grow from real failures. Keep it separate from any examples placed in prompts.
- Score deterministically wherever possible (enums, dates, IDs, schema validity). Use an LLM judge only for free text, and calibrate it against human labels.
- Report rates, not anecdotes: schema-valid, semantic-valid, per-field accuracy, retry and fallback rate, latency, cost.
- Reduce noise: low temperature, repeat gating runs, and compare against the previous run with a margin.
- Include adversarial cases (prompt injection, empty, huge, multilingual) and assert safety properties, such as "no state change without approval".
- Never edit expected outputs to make a run pass without a recorded reason.

## Flaky tests
Do not add blind retries. Find the cause (time, ordering, shared state, async race). If it cannot be fixed immediately, quarantine it with a tracking issue and a deadline.

## Checklist before finishing a change
- [ ] Bug fixes include a regression test that failed first.
- [ ] Cross-user, failure-injection and boundary cases covered.
- [ ] No network, wall-clock or unseeded randomness in tests.
- [ ] AI changes: fake-client tests updated and an eval run attached.
- [ ] Tests pass in random order and in parallel.
