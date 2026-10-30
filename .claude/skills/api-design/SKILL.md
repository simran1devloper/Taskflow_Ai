---
name: api-design
description: Standards for designing REST/JSON HTTP APIs - resource naming, status codes, error envelope, pagination, idempotency, versioning, auth and compatibility rules. Use when adding or changing any endpoint, request/response schema, or error behavior.
---

# API Design

Reusable API conventions. Specific endpoints are in `plans/`. The API is a contract with the frontend and any future client, so changes follow compatibility rules below.

## Resources and verbs
- Nouns, plural, lowercase, kebab-case where multi-word: `/tasks`, `/tasks/{id}`, `/projects/{id}/tasks`.
- Standard methods: `GET` read, `POST` create, `PATCH` partial update, `PUT` full replace (use sparingly), `DELETE` remove.
- Actions that are not CRUD use a sub-resource verb: `POST /tasks/{id}/complete`. Do not tunnel actions through `PATCH` flags.
- Nest at most one level. Beyond that, use query filters.
- All routes are versioned under `/api/v1`.

## Status codes
| Code | Use |
|---|---|
| 200 / 201 / 204 | Success / created / success with no body |
| 202 | Accepted for asynchronous processing |
| 400 | Malformed request |
| 401 | Not authenticated |
| 403 | Authenticated but not allowed (own-resource checks return 404 instead) |
| 404 | Not found, including other users' resources |
| 409 | State conflict (stale version, already applied) |
| 410 | Permanently gone (for example an expired proposal) |
| 422 | Well-formed but fails validation |
| 429 | Rate limited (include `Retry-After`) |
| 5xx | Server fault; never include internals |

## Error envelope
One shape everywhere:
```json
{ "error": { "code": "validation_failed", "message": "Human readable", "details": [{"field": "due_at", "issue": "invalid_date"}], "request_id": "..." } }
```
- `code` is stable and machine-readable; `message` may change. Clients branch on `code`.
- Never leak stack traces, SQL, file paths or secrets.

## Requests and responses
- JSON, `snake_case` field names, ISO 8601 UTC timestamps, string UUIDs.
- Define separate request and response schemas per resource (`TaskCreate`, `TaskUpdate`, `TaskRead`); never expose ORM models or internal columns.
- Reject unknown fields on writes so typos fail loudly.
- Partial updates: only provided fields change; `null` means clear the value only where the field is nullable and documented.
- Return the updated resource from create/update to save a round trip.

## Collections
- Cursor pagination (`limit`, `cursor`); response `{items, next_cursor}`. Enforce a maximum `limit`.
- Filtering through explicit query parameters; sorting through `sort=field` / `sort=-field` from an allow-list.
- A list response is stable for the same query and cursor.

## Idempotency and concurrency
- `GET`, `PUT`, `DELETE` are idempotent by definition. Make `POST` actions that create or apply things idempotent via an `Idempotency-Key` header; replay returns the original response.
- Use a `version` field (or ETag with `If-Match`) for optimistic concurrency; mismatch returns 409 with the current version.

## Long-running and AI operations
- If an operation may exceed a few seconds, return 202 with a status resource instead of holding the connection.
- Operations that propose changes return the proposal; applying it is a separate explicit call.

## Security
- Authenticate every route by default; opt-out must be explicit and reviewed.
- Authorize at the service layer on every access, not only at the route.
- Validate and bound all inputs (length, range, enum, list size).
- Rate limit per user and per IP; stricter on auth and expensive endpoints.
- State-changing cookie-authenticated requests require CSRF protection.
- Set CORS to specific origins; never `*` with credentials.

## Compatibility rules
- **Non-breaking:** adding optional request fields, adding response fields, adding endpoints, adding enum values *only if* clients are documented to tolerate unknown values.
- **Breaking:** removing/renaming fields, changing types or semantics, tightening validation, changing status codes. Requires a new version or a deprecation window.
- The OpenAPI document is generated from code and committed/snapshotted; CI diffs it and flags breaking changes.

## Checklist before finishing a change
- [ ] Naming, method and status codes follow the tables above.
- [ ] Request/response schemas are separate and documented in OpenAPI.
- [ ] Errors use the envelope with a stable `code`.
- [ ] Idempotency, pagination and concurrency handled where relevant.
- [ ] Authorization and cross-user access tested.
- [ ] Breaking-change check passes, or a version bump is planned.
