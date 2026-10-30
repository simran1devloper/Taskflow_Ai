---
name: documentation
description: How to write and maintain project documentation - ADRs, feature plans, API docs, runbooks, READMEs, and code comments - including when to write each, the structure to use, and how to keep docs from going stale. Use when recording a decision, writing or updating a plan or runbook, or changing behavior that existing docs describe.
---

# Documentation

Documentation exists to let someone make a correct decision or take a correct action without asking the author. Write only what serves that, and keep it where it will be maintained.

## Where things live
| Need | Home | Changes |
|---|---|---|
| Why a significant technical choice was made | `architecture/decisions/ADR-NNN-*.md` | Immutable once accepted; supersede with a new ADR |
| What a feature will do and how it will be built | `plans/NN-*.md` | Updated as work proceeds; completion status maintained |
| Project-wide roadmap, phases, open questions | `PLAN.md` | Updated at phase boundaries |
| Rules for AI assistants and contributors | `CLAUDE.md` | Short, durable, command- and rule-focused |
| Reusable engineering practice | `.claude/skills/*/SKILL.md` | Updated when practice changes |
| API behavior | OpenAPI generated from code | Never hand-maintained |
| Operating the system | `docs/runbook.md` | Updated with every operational change |
| Setup and orientation | `README.md` | Verified by following it on a clean machine |
| Why a line of code is the way it is | Code comment | Next to the code |

Do not duplicate: link to the single source instead. Duplicates drift.

## ADRs (Architecture Decision Records)
Write one when a decision is hard to reverse, affects multiple modules, or someone will later ask "why did we do it this way?". Not for routine choices.

Structure:
```
# ADR-NNN: Title (a decision, not a topic)
- Status: Proposed | Accepted | Superseded by ADR-XXX
- Date:
## Context        forces and constraints; facts, not advocacy
## Open question  anything unresolved that blocks acceptance (remove when resolved)
## Decision       what we will do, specific enough to act on
## Alternatives considered   each with the reason it was rejected
## Consequences   good, bad and neutral
## Revisit when   concrete triggers that would reopen the decision
```
- One decision per ADR. Number sequentially; never renumber.
- Accepted ADRs are not edited except for status and links. A changed decision gets a new ADR that supersedes the old one.

## Feature plans
Use the standard sections: objective and scope, functional requirements, technical design, component interactions, API contracts, database changes, implementation subtasks, failure scenarios, testing strategy, acceptance criteria, dependencies and status.
- Write the plan before building; update it when reality diverges rather than leaving it wrong.
- Acceptance criteria must be checkable (a command, a test, an observable behavior).
- Mark unresolved product or technical questions explicitly (for example **[OPEN]**) with what they block.
- Keep status honest: percent or checklist updated in the same PR as the work.

## Runbooks
For each operational task (deploy, rollback, restore, rotate a secret, handle a provider outage) give: when to use it, prerequisites, exact commands, how to verify success, and how to back out. Write for someone stressed at 3 a.m. who did not build the system. Test them: perform the procedure from the document.

## README
Purpose in two sentences; prerequisites; the shortest path to running the app and tests; project layout; links to plans, ADRs and runbook. If the quick-start steps fail on a clean machine, that is a bug.

## Code comments and docstrings
- Explain *why*, constraints, and non-obvious consequences. Do not narrate what the code visibly does.
- Document public service interfaces: purpose, arguments, errors raised, side effects.
- Delete comments that no longer match the code. Reference an issue instead of leaving a bare TODO.

## Writing style
- Lead with the conclusion or the action. Short sentences, concrete nouns, consistent terms.
- Prefer tables for comparisons, numbered lists for procedures, and diagrams (Mermaid) for flows and relationships. Keep diagrams small and text-diffable.
- State assumptions and unknowns plainly; never present a guess as a fact.
- Include examples with real values for formats and commands.
- Use relative links and file paths so they stay valid.

## Keeping docs current
- A change that alters documented behavior updates the doc in the same PR; reviewers check for it.
- Generate what can be generated (OpenAPI, schema diagrams, CLI help).
- Date-stamp decisions and status; remove dead docs rather than leaving them to mislead.

## Checklist before finishing a change
- [ ] Hard-to-reverse decision has an ADR (or an update to a proposed one).
- [ ] Affected plan's status and acceptance criteria updated.
- [ ] Operational impact reflected in the runbook.
- [ ] README commands still work.
- [ ] No duplicated facts; links point to the single source.
