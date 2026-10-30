# ADR-002: Database Selection

- **Status:** Proposed (blocked on an open question, see below)
- **Date:** 2026-10-30

## Context

The data is relational, and AI features need to store semi-structured payloads (LLM tool-call arguments, proposals, agent state). Background AI jobs may write concurrently with API requests.

## Open question (required before acceptance)

Will TaskFlow AI be **single-user and local-first**, or **multi-user and hosted** (now or in the near future)? The decision below assumes multi-user/hosted. If the answer is single-user local-first, SQLite becomes the better choice.

## Decision (provisional)

Use **PostgreSQL** for development and production, run via Docker Compose, accessed through **SQLAlchemy** with **Alembic** migrations.

- Use one engine everywhere. No SQLite in dev with Postgres in prod.
- Use `JSONB` for LLM payloads and proposal data.
- If a job queue is needed, start with Postgres (`SELECT … FOR UPDATE SKIP LOCKED`) before adding Redis.
- `pgvector` is available later if semantic search is added.

## Alternatives considered

- **SQLite:** zero setup and ideal for single-user local use. Rejected provisionally because of its single-writer limit (`database is locked` risk with background jobs), limited `ALTER TABLE`, loose typing, and awkward hosted deployment.
- **SQLite for dev, Postgres for prod:** rejected. Dialect differences (JSON, locking, migrations, case sensitivity) mean tests can pass on one engine and fail on the other.

## Consequences

- Real concurrency, transactional DDL, strict constraints, and dev/prod parity.
- Requires running a database server (one Compose service).
- Migrating from SQLite later would be harder than starting on Postgres.
