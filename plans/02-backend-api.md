# 02 — Backend API

- **PLAN.md phase:** 1
- **Status:** Not started
- **Blocked by:** PLAN.md open questions 1 and 4

## 1. Objective and scope
Provide the HTTP layer, auth, error model and cross-cutting behavior shared by every endpoint.
**In scope:** auth, API conventions, error format, pagination, idempotency, rate limiting.
**Out of scope:** task and proposal endpoints (plans 03 and 07 define them).

## 2. Functional requirements
- Register, log in, log out, fetch the current user. **[OPEN]** If single-user local, replace with a local session or none.
- Every route except register and login requires authentication.
- Consistent JSON error shape across the whole API.
- OpenAPI schema is the source for the frontend client.

## 3. Technical design
- FastAPI routers per module under `/api/v1`. Routers are thin: parse, call the service, map the result.
- Auth: Argon2 password hashing; short-lived access token in an `HttpOnly`, `Secure`, `SameSite=Lax` cookie, plus CSRF protection for state-changing routes. Cookie-based because the client is a first-party SPA.
- `current_user` dependency injected into every service call so `user_id` filtering cannot be forgotten.
- Domain errors (`NotFound`, `ValidationFailed`, `Conflict`, `Forbidden`) mapped to HTTP in one exception handler.
- Pagination: cursor-based (`limit`, `cursor`). Idempotency: `Idempotency-Key` header on POSTs that create or apply.
- Rate limiting per user and per IP; stricter on auth and AI routes.

## 4. Component interactions
Request → request-ID middleware → auth dependency → router → module service → repository → DB. Errors bubble to the global handler, which logs once and returns the standard envelope.

## 5. API contracts
Error envelope:
```json
{ "error": { "code": "validation_failed", "message": "...", "details": [], "request_id": "..." } }
```
| Method | Path | Body → Response |
|---|---|---|
| POST | `/api/v1/auth/register` | `{email, password}` → 201 `User` |
| POST | `/api/v1/auth/login` | `{email, password}` → 200 `User` + cookie |
| POST | `/api/v1/auth/logout` | → 204 |
| GET | `/api/v1/auth/me` | → 200 `User` |

Status codes: 400 malformed, 401 unauthenticated, 403 forbidden, 404 not found (also used for other users' resources), 409 conflict, 422 validation, 429 rate limited.

## 6. Database changes
Uses `users` from plan 01. Adds a `sessions` or token-revocation table only if logout must invalidate tokens server-side (decide in ADR-005).

## 7. Implementation subtasks
1. Error types, handler, envelope.
2. Request-ID and access-log middleware.
3. Password hashing and `auth` module service.
4. Cookie auth, CSRF, `current_user` dependency.
5. Pagination and idempotency helpers.
6. Rate limiter (in-process first; move to Postgres or Redis only if multi-instance).
7. Export OpenAPI to `openapi.json` in CI to detect breaking changes.

## 8. Failure scenarios
- Brute-force login: rate limit plus uniform error message and timing.
- Duplicate POST on retry: idempotency key returns the original response.
- Resource of another user: always 404, never 403, to avoid leaking existence.
- Unhandled exception: 500 with request ID only; stack trace goes to logs, not the client.

## 9. Testing strategy
Integration tests for each auth flow, cross-user access tests on every router (parametrized), error envelope contract tests, rate-limit tests, and an OpenAPI snapshot diff in CI.

## 10. Acceptance criteria
- [ ] Unauthenticated access to any protected route returns 401.
- [ ] A user cannot read or modify another user's data (tested for every resource).
- [ ] All errors use the envelope.
- [ ] OpenAPI is generated and the frontend client builds from it.

## 11. Dependencies and completion status
- Depends on: 00, 01.
- Unblocks: 03, 04, 07.
- Completion: 0%.
