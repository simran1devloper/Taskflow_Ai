---
name: code-review
description: Project review checklist for TaskFlow AI changes - architecture boundaries, security, data integrity, AI safety, tests, and how to give actionable feedback. Use when reviewing a diff, PR or branch for this repo, or self-reviewing before a commit.
---

# Code Review (TaskFlow AI)

A project-specific review lens. It complements, and does not replace, any general-purpose review tooling. Review in order: **correctness and safety first, design second, style last** (style belongs to the formatter and linter).

## Process
1. Read the plan or ADR the change claims to implement. Confirm the diff matches its scope; flag unrelated changes.
2. Read the tests first. Do they describe the intended behavior? What would still pass if the code were wrong?
3. Read the code against the checklists below.
4. Run the checks locally if anything looks risky (tests, migrations on an empty DB, linters).
5. Write findings using the format at the end.

## Architecture
- Module boundaries respected: no imports of another module's `models`/`repository`; boundary linter passes.
- Business logic is in services or pure functions, not routers or components.
- No new abstraction, dependency or service without a stated need. Question anything that adds a framework, queue or agent where a function would do (see ADR-001 and ADR-003).
- Decisions that change an ADR come with an updated or new ADR.

## Security
- Every user-owned query filters by user; new endpoints have cross-user tests.
- Inputs validated and bounded; outputs use response schemas, not ORM objects.
- No secrets, tokens or personal data in code, logs, fixtures or error messages.
- No string-built SQL; no `dangerouslySetInnerHTML` or unescaped untrusted content.
- Auth, CSRF, CORS and rate limits unchanged or deliberately updated.
- New dependencies: necessary, maintained, pinned, and audit-clean.

## Data integrity
- Multi-step writes are in one transaction; no external call inside it.
- Constraints enforce invariants in the database.
- Migration is expand/contract-safe, applies from empty, and shows no model drift.
- Retries and double submits are safe (idempotency, unique constraints, locks).
- Time handling is UTC in storage and explicit about time zones at the edges.

## AI-specific
- The model has no write access; writes go through proposals approved by the user and applied by deterministic code.
- LLM output is validated (schema + semantic checks) before use and again at apply time.
- Untrusted text (task content, imports) is treated as data in prompts, never as instructions with authority.
- Retries are bounded; failure has a defined fallback and never loses user input or writes partially.
- Prompt/model/version recorded on runs; a prompt or model change ships with an eval result.
- Cost and rate limits considered for new LLM calls.

## Tests
- New behavior and each bug fix has a test; bug fixes have a regression test.
- Failure paths, boundaries, and authorization are tested, not only the happy path.
- No flaky patterns: sleeps, wall-clock time, shared state, order dependence.
- Tests assert behavior, not implementation details.

## Maintainability
- Names say what things are; functions do one thing; no dead code or commented-out blocks.
- Comments explain *why*; no stale TODOs without an issue.
- Public interfaces are small and typed.
- Errors are specific and actionable.

## Performance and operability
- No N+1 queries or unbounded lists/loops; large work moves to background jobs.
- Logs are structured, include correlation IDs, and contain no sensitive data.
- New failure modes are observable (log, metric or alert) and documented in the runbook if operational.

## Giving feedback
Use severity labels so the author knows what blocks:
- **blocker:** bug, security issue, data loss/corruption risk, broken boundary, missing test for risky behavior.
- **should-fix:** maintainability or design problem worth fixing before merge.
- **nit:** optional polish; never block on these.
- **question:** you need context to judge.

Each finding states: location, the problem, the concrete failure scenario (inputs/state leading to wrong behavior), and a suggested fix. Prefer "this fails when X because Y" over "this looks wrong". Note what is good too. Do not restate what the formatter or linter will catch.

## Approval criteria
Approve when there are no blockers, should-fix items are addressed or consciously deferred with an issue, tests and CI pass, and the change matches its plan.
