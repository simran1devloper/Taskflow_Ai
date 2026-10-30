# 00 — Project Foundation

- **PLAN.md phase:** 0
- **Status:** Not started
- **Blocked by:** PLAN.md open questions 1 (single vs multi-user), 4 (stack ADR-005), 6 (deployment target)

## 1. Objective and scope
Create a repo skeleton where every later feature lands with CI, tests, migrations and boundary checks already in place.
**In scope:** repo layout, tooling, Compose, CI, pre-commit hooks, config, logging setup, durable `CLAUDE.md`.
**Out of scope:** any domain logic.

## 2. Functional requirements
- One command starts the full dev stack (`docker compose up`).
- One command runs lint, types, tests and boundary checks (`make check`).
- Config comes from environment variables, validated at startup.
- App fails fast with a clear message on missing or invalid config.

## 3. Technical design
```
backend/
  app/
    main.py            # FastAPI factory
    core/              # config, logging, db session, errors
    modules/           # auth, projects, tasks, proposals, audit, ai_orchestration
  alembic/
  tests/{unit,integration,evals}/
frontend/              # created in plan 04
docker-compose.yml
Makefile
.github/workflows/ci.yml
```
- Python 3.12, `uv` for dependencies, `ruff` (lint + format), `mypy --strict` on `app/`, `pytest`.
- `pydantic-settings` for config. `structlog` for JSON logs.
- `import-linter` contracts: modules may import only each other's `service` and `schemas`, never `models` or `repository`.

## 4. Component interactions
CI runs: ruff, mypy, import-linter, pytest (Postgres service container), `alembic upgrade head` on a clean DB, then `alembic check` for model/migration drift.

## 5. API contracts
Only `GET /healthz` (liveness) and `GET /readyz` (DB reachable). Both return `{"status": "ok"}`.

## 6. Database changes
Initialise Alembic with an empty baseline migration. No tables.

## 7. Implementation subtasks
1. Replace `CLAUDE.md` content with durable rules (stack, commands, boundaries, "LLM never writes").
2. `pyproject.toml`, `uv.lock`, tool configs.
3. Docker Compose: `db` (Postgres), `api`. Healthchecks on both.
4. App factory, settings, structured logging with request-ID middleware.
5. Alembic setup and baseline.
6. `import-linter` config with a deliberately failing fixture to prove it works.
7. CI workflow and pre-commit config.
8. `Makefile` targets: `up`, `check`, `test`, `migrate`.

## 8. Failure scenarios
- Missing env var: startup aborts listing the variable name, never its value.
- DB down at startup: `/readyz` fails, `/healthz` still passes.
- Migration drift: CI fails on `alembic check`.

## 9. Testing strategy
Smoke tests for `/healthz` and `/readyz`; a config-validation unit test; CI proves the boundary linter fails on a violating import.

## 10. Acceptance criteria
- [ ] Fresh clone to running API in under 5 minutes using README steps.
- [ ] CI green on the empty app.
- [ ] Boundary violation fails CI.
- [ ] No secrets in repo; `.env.example` present.

## 11. Dependencies and completion status
- Depends on: none.
- Unblocks: all other plans.
- Completion: 0%.
