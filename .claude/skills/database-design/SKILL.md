---
name: database-design
description: Rules for designing relational schemas and writing safe migrations with PostgreSQL, SQLAlchemy and Alembic - keys, constraints, indexes, JSONB use, expand/contract migrations, locking, and tenant isolation. Use when adding or changing a table, column, index, query pattern, or migration.
---

# Database Design

Reusable schema and migration practice. The concrete tables are in `plans/01-database-design.md`; the engine choice is in `architecture/decisions/ADR-002`.

## Schema principles
- **Let the database enforce invariants.** Use `NOT NULL`, `UNIQUE`, `FOREIGN KEY` and `CHECK` constraints; application validation is a second layer, not the only one.
- Model real relationships as columns and foreign keys. Use JSONB only for genuinely variable or opaque payloads (LLM output, rule definitions), and never for data you filter, join or constrain on regularly.
- Prefer `text` + `CHECK` over native enum types for status-like fields; adding a value is a cheap constraint change, not a type migration.
- UUID primary keys (time-ordered where possible). `timestamptz` in UTC for all times. `created_at`/`updated_at` on every table.
- Soft delete (`deleted_at`) only where history must remain referable; hard delete otherwise. Pick per table and document it.
- Name things consistently: plural snake_case tables, `<entity>_id` foreign keys, `ix_`/`uq_`/`ck_` prefixes via a naming convention set in the metadata so Alembic generates stable names.

## Multi-tenant (per-user) data
- Every user-owned table has `user_id NOT NULL` with an index that leads with it.
- Repositories take `user_id` as a required argument; there is no "get by id" without it.
- Consider composite foreign keys (`(user_id, project_id)`) when you must guarantee a child cannot reference another user's parent.
- Audit and log tables are append-only: no UPDATE/DELETE privileges for the application role.

## Indexes
- Add an index for a specific query, written next to that query. Check it with `EXPLAIN (ANALYZE, BUFFERS)` on realistic row counts.
- Column order follows equality filters first, then range/sort columns.
- Use partial indexes for hot subsets (for example only `status = 'pending'`).
- Each index costs writes and space; remove ones that go unused.

## Concurrency
- Use `SELECT ... FOR UPDATE` for check-then-act on a row; `SKIP LOCKED` for queue-style claiming.
- Use a `version` column for optimistic locking on user-edited entities.
- Keep transactions short, and never hold one across a network call.
- Unique constraints, not "check then insert", prevent duplicates.

## Migrations (Alembic)
- One logical change per revision; revisions are immutable once merged.
- **Expand/contract** for anything touching live data:
  1. Expand: add the new column/table as nullable or with a default; deploy code that writes both.
  2. Backfill in batches (outside the DDL transaction for large tables).
  3. Switch reads; then contract in a later release: drop the old column.
- Never rename or drop in the same release the code still uses it.
- Adding `NOT NULL` or a foreign key to a big table: add as `NOT VALID`/nullable first, then validate.
- Create indexes with `CONCURRENTLY` on non-empty tables (this requires running outside a transaction).
- Every migration must upgrade from an empty database in CI and `alembic check` must show no model drift. Provide a downgrade only if it is genuinely safe; otherwise document forward-fix.
- Data migrations are separate revisions from schema migrations.

## SQLAlchemy usage
- 2.x style: typed `Mapped[...]` models, `select()` statements.
- Load related data explicitly (`selectinload`/`joinedload`); no lazy loading in request paths.
- Keep ORM models out of API responses; map to Pydantic schemas.
- Tests use the same engine as production (no SQLite substitute).

## JSONB guidance
- Store the version of the schema/prompt that produced a payload next to it.
- Validate on write with Pydantic; do not trust stored shape on read without a version check.
- Index with GIN only for queries you actually run.

## Checklist before finishing a change
- [ ] Constraints enforce the invariants, not just app code.
- [ ] `user_id` present, indexed and required by the repository.
- [ ] Index justified by a named query and verified with EXPLAIN.
- [ ] Migration is expand/contract-safe, applies from empty, and `alembic check` is clean.
- [ ] Concurrency case considered (lock, version, or unique constraint).
