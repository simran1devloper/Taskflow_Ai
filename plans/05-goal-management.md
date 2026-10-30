# 05 — Goal Management

- **PLAN.md phase:** 7+ (post-MVP)
- **Status:** Not started, not in the MVP in PLAN.md
- **Blocked by:** product decision: are goals a separate entity from projects? **[OPEN]**

## 1. Objective and scope
Let users define goals with target dates and link tasks to them, so progress is visible and the AI planner can break a goal into tasks.
**In scope:** goals CRUD, linking tasks, progress calculation, goal → subtasks AI breakdown (via plan 07).
**Out of scope:** OKR hierarchies, shared goals, habit tracking.

## 2. Functional requirements
- Create, edit, archive goals with title, description, target date, status (`active`, `achieved`, `abandoned`).
- Link and unlink tasks; a task may belong to at most one goal.
- Progress = completed linked tasks / total linked tasks (deterministic).
- "Break down this goal" produces a proposal containing suggested tasks (plan 07).

## 3. Technical design
- New `goals` module with the same layout as `tasks`.
- Tasks reference goals through `goal_id` (nullable FK). The `goals` module exposes `get_goal`, `link_task`; `tasks` calls only that interface.
- Progress is computed on read from a query, not stored, to avoid drift. Cache later if needed.
- Consider whether goals should simply be a project type before building a new module. If yes, this plan shrinks to a flag on `projects`.

## 4. Component interactions
Router → GoalService → TaskService (read counts). Goal breakdown: GoalService supplies context to `ai_orchestration`, which returns a proposal of `create_task` actions with `goal_id` set.

## 5. API contracts
| Method | Path | Notes |
|---|---|---|
| GET/POST | `/api/v1/goals` | list, create |
| GET/PATCH | `/api/v1/goals/{id}` | includes `progress: {done, total}` |
| POST | `/api/v1/goals/{id}/tasks` | `{task_ids: []}` link |
| DELETE | `/api/v1/goals/{id}/tasks/{task_id}` | unlink |
| POST | `/api/v1/goals/{id}/breakdown` | creates a proposal, returns 202 with proposal id |

## 6. Database changes
New `goals` table (id, user_id, title, description, target_date, status). Add nullable `tasks.goal_id` with index. Expand/contract migration; no data backfill.

## 7. Implementation subtasks
1. Decide goals vs project type (ADR).
2. Migration, models, service, routes.
3. Progress query and tests.
4. Frontend goal views (extends plan 04).
5. Breakdown prompt and eval cases (in plan 07/09).

## 8. Failure scenarios
- Linking another user's task: 404.
- Deleting a goal with tasks: tasks are kept, `goal_id` set to null.
- Breakdown proposes duplicate existing tasks: semantic check flags near-duplicates in the proposal.

## 9. Testing strategy
Unit tests for progress math (including zero tasks), integration tests for link/unlink and isolation, eval set for breakdown quality.

## 10. Acceptance criteria
- [ ] Goals CRUD and task linking work with isolation guaranteed.
- [ ] Progress is correct for all states, including empty.
- [ ] Breakdown yields a reviewable proposal and never writes directly.

## 11. Dependencies and completion status
- Depends on: 03, 07.
- Unblocks: 06 (goal metrics).
- Completion: 0%.
