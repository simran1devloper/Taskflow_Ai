---
name: backend-development
description: Conventions for writing Python/FastAPI backend code in a modular monolith - module layering, transactions, error handling, config, logging, async rules. Use when adding or changing any backend module, service, repository, or background job.
---

# Backend Development

Reusable rules for backend code. Feature behavior lives in `plans/`; technical decisions live in `architecture/decisions/`. This file is about *how* to write the code.

## Module layering
Each module under `app/modules/<name>/` has the same shape:

| File | Responsibility | May import |
|---|---|---|
| `router.py` | Parse request, call service, map result. No logic. | own `service`, `schemas` |
| `service.py` | Business rules, transaction boundary, public interface | own `repository`, other modules' `service`/`schemas` |
| `repository.py` | Queries only. Every query takes `user_id` when the data is user-owned. | own `models` |
| `models.py` | ORM tables | SQLAlchemy |
| `schemas.py` | Pydantic IO types, also the cross-module contract | Pydantic |

Rules:
- Never import another module's `models` or `repository`. If you need its data, add a method to its `service`.
- Dependencies point inward: router → service → repository. Never the reverse.
- Keep pure logic (state machines, recurrence, scoring) in plain functions with no I/O, so it is unit-testable without a DB.

## Transactions
- The **service** owns the transaction. Repositories never commit.
- One business operation = one transaction. If an operation writes data and an audit entry, both go in the same transaction.
- Do not call external services (LLM, email, HTTP) while holding a transaction open. Call first, then open the transaction to write the result.
- Use `SELECT ... FOR UPDATE` for read-check-write sequences on contended rows; use a `version` column for user-facing optimistic concurrency.
- Make state-changing endpoints idempotent (idempotency key or natural key) because clients retry.

## Errors
- Raise domain exceptions (`NotFound`, `Conflict`, `ValidationFailed`, `Forbidden`) from services. Translate them to HTTP in exactly one handler.
- Never catch `Exception` to hide a failure. Catch only what you can handle, and re-raise with context otherwise.
- Another user's resource is `NotFound`, not `Forbidden`.
- Error messages for clients carry a stable `code`; log details server-side with the request ID.

## Configuration and secrets
- All config through one typed settings object, validated at startup. Fail fast with the variable *name* (never the value).
- No `os.environ` reads outside the settings module.
- Secrets never appear in code, logs, error messages, or test fixtures.

## Logging
- Structured JSON logs. Bind `request_id`, `user_id` and the relevant entity ID once per request via context vars.
- Log an event at the layer that handles it, once. Do not log-and-rethrow at every layer.
- Never log passwords, tokens, or full user content. Truncate or hash user-provided text.

## Async and performance
- Choose sync or async per call path and stay consistent. Do not call blocking I/O inside `async def`.
- Avoid N+1 queries: load relations explicitly, and check query counts in tests for list endpoints.
- Anything slower than ~1 s, or dependent on an external API, belongs in a background job, not the request.
- Add an index with the query that needs it, and verify with `EXPLAIN` on realistic data.

## Typing and style
- `mypy --strict` on application code. No untyped public functions.
- Pydantic models at every boundary (HTTP, LLM output, job payloads). Plain dataclasses inside the domain.
- Prefer small functions and explicit names over comments. Comment *why*, not *what*.

## Checklist before finishing a change
- [ ] Layering respected; boundary linter passes.
- [ ] Transaction scope is correct and no external call is inside it.
- [ ] Every user-owned query filters by user.
- [ ] New config is in the settings object and `.env.example`.
- [ ] Errors are domain exceptions; no swallowed exceptions.
- [ ] Tests cover the success path, a failure path, and cross-user access.
