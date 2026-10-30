# 06 — Productivity Analytics

- **PLAN.md phase:** 7+ (post-MVP)
- **Status:** Not started, not in the MVP in PLAN.md
- **Blocked by:** product decision on which metrics matter **[OPEN]**

## 1. Objective and scope
Show users how they work: throughput, overdue trends, completion by project. Entirely deterministic SQL; no LLM in the metric path.
**In scope:** a small set of metrics, a dashboard API, optional AI-written summary of the numbers (read-only).
**Out of scope:** time tracking, team analytics, predictive models.

## 2. Functional requirements
- Tasks completed per day/week over a selectable range.
- Overdue count and average age of open tasks.
- Completion rate per project and per priority.
- Optional "weekly summary" in plain language, generated from the computed numbers only (no raw task text unless the user opts in).

## 3. Technical design
- Metrics computed by SQL over `tasks` and `audit_log` (completion events). No separate analytics store at MVP scale.
- Add `completed_at` on tasks (set in the transition code) so metrics do not rely on audit parsing.
- Responses are computed per request with an index-backed query; add a materialized view or cache only if a measured p95 is too high.
- Summary: an LLM step that receives the metric JSON and returns text; read-only, so no approval is needed (ADR-004). Output must not state numbers absent from the input; a deterministic check compares numbers in the text against the input.

## 4. Component interactions
Router → AnalyticsService → read-only queries via `tasks` service interface (no direct table reads, per ADR-001). Summary: AnalyticsService → `ai_orchestration.summarize(metrics)`.

## 5. API contracts
| Method | Path | Response |
|---|---|---|
| GET | `/api/v1/analytics/throughput?from&to&bucket=day\|week` | `[{bucket, completed}]` |
| GET | `/api/v1/analytics/overdue` | `{count, avg_age_days}` |
| GET | `/api/v1/analytics/completion-rate?group_by=project\|priority` | `[{key, done, total}]` |
| POST | `/api/v1/analytics/summary` | `{range}` → `{text, metrics}` |

## 6. Database changes
Add `tasks.completed_at` (nullable, backfilled from audit where available) and an index on `(user_id, completed_at)`.

## 7. Implementation subtasks
1. `completed_at` migration and transition hook.
2. Query functions exposed via the `tasks` service interface.
3. Analytics module, routes, time-zone handling for buckets.
4. Frontend charts (extends plan 04; use the `dataviz` skill).
5. Summary prompt, number-consistency check, eval cases.

## 8. Failure scenarios
- Empty data: return zeros, not errors; UI shows an empty state.
- Large ranges: cap at 1 year and enforce the limit server-side.
- LLM summary contradicts the numbers: the check fails, return metrics with no summary.

## 9. Testing strategy
Seeded fixtures with known answers; time-zone and bucket-boundary tests; consistency-check unit tests; load test at 100k tasks per user to confirm p95 target.

## 10. Acceptance criteria
- [ ] Metrics match hand-computed values on fixtures.
- [ ] p95 under 300 ms at 100k tasks per user.
- [ ] Summary never includes numbers not in its input.

## 11. Dependencies and completion status
- Depends on: 03, 04; 05 optionally; 07 for the summary.
- Unblocks: nothing.
- Completion: 0%.
