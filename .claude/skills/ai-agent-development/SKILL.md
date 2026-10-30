---
name: ai-agent-development
description: Engineering practice for LLM features and agents - deciding workflow vs agent, structured output, validation and retry, tool permissions, human approval, prompt injection defense, context management, observability and evals. Use when adding or changing any LLM call, prompt, tool, or agent behavior.
---

# AI / Agent Development

Reusable practice for building reliable LLM features. Project decisions: `architecture/decisions/ADR-003` (orchestration) and `ADR-004` (human approval). Feature specifics: `plans/07` and `plans/08`.

## 1. Decide the simplest mechanism first
Walk down this list and stop at the first that works:
1. **Plain code.** Anything with deterministic rules (CRUD, filtering, scheduling, permissions, math).
2. **Single LLM call with structured output.** Extraction, classification, rewriting, summarizing, drafting.
3. **Fixed multi-step workflow.** Several LLM/code steps in a predetermined order, each independently testable.
4. **Agent loop.** The model chooses the next tool call. Use only when the number or order of steps cannot be known in advance *and* evals show steps 2-3 fail.
5. **Multiple agents.** Only for real trust-boundary isolation, real parallelism, or measured specialist gains. Each handoff loses context and adds failure modes.

Moving down a level requires evidence (eval results), not intuition. Record the decision in an ADR.

## 2. Treat the model as an unreliable, untrusted component
- Its output is *input to your system*: validate it like user input.
- It can be wrong, malformed, inconsistent between runs, or manipulated by text it reads.
- Build so that the worst-case output causes a rejected proposal, not damage.

## 3. Structured output and validation
- Define the output as a Pydantic schema; request structured output from the provider (JSON schema / tool-call forcing) rather than parsing free text.
- Validate in two layers:
  1. **Schema** - types, enums, required fields, lengths.
  2. **Semantic** - referenced IDs exist and belong to the user; dates are plausible; values are consistent with the current data.
- On failure: retry a bounded number of times (default 2), passing the specific validation error back. Then take a defined fallback. Never loop unbounded, never partially apply.
- Keep schemas small and flat. Large nested schemas fail more often.

## 4. Tools and permissions
- Allow-list tools. Each declares name, input schema, a `read_only` flag, and the permission it needs.
- **Read tools** may be called by the model directly, scoped to the current user and bounded in result size.
- **Write capability is never given to the model.** A "write tool" emits a *proposed action*; deterministic code applies it after approval.
- Tool arguments are validated like any API input, including ownership checks. Reject unknown tools.
- Principle of least privilege: expose only the tools a given step needs.

## 5. Human approval
- Any AI output that would change user data is a **proposal**: stored, shown with a clear diff, and applied only after explicit user approval.
- Re-validate at apply time; data may have changed. Stale or expired proposals are rejected.
- Apply atomically with an audit record (actor, proposal ID, model and prompt version).
- Relax approval (for example auto-apply) only for low-risk, reversible actions, with undo and eval evidence.

## 6. Prompt injection defense
Assume any text the model reads (task titles, notes, imported content) may contain instructions.
- Structural defense beats prompt wording: no write access, allow-listed tools, user approval, validation.
- Put untrusted content in clearly delimited data sections and say it is data; do not rely on that alone.
- Do not let untrusted content influence which tools are available or what privileges they have.
- Never put secrets in prompts. Do not let the model emit URLs/markup that the UI renders or fetches automatically.
- Maintain adversarial eval cases and assert the safety property (no unapproved state change).

## 7. Context and memory
- Build context deliberately: only what the step needs, within a token budget. Retrieve by relevance, cap by count and tokens, and record when truncation occurred.
- Default to stateless requests. Add conversational memory only when a feature demands it, as a short rolling window or summary with an expiry.
- Long-term memory about the user is a product and privacy decision; require explicit design and user control before adding it.
- Include current date, time zone and locale explicitly; the model does not know them.

## 8. Prompts
- Store prompts as versioned files in the repo, not inline strings. Record the version on every run.
- Keep instructions specific, with the output schema, a few short examples, and explicit handling for ambiguity ("return an error field rather than guess").
- Examples in prompts must not overlap with eval cases.
- Change one thing at a time; every prompt or model change ships with an eval comparison against the previous version.

## 9. Reliability
- Wrap the provider behind an interface (`LLMClient`) with timeouts, bounded retries with backoff for transient errors, and a circuit breaker.
- Define degraded behavior up front: the non-AI path must keep working when the provider is down.
- Enforce per-user rate limits and cost caps; cap input and output size.
- Make calls idempotent where possible and never hold a database transaction open during a call.
- Never lose user input because the AI step failed.

## 10. Observability
Log per run: feature, model, prompt version, input size, raw output (subject to the data-retention policy), validation errors, retry count, fallback taken, latency, tokens/cost, and the outcome of the resulting proposal (approved/edited/rejected/expired).
Track: schema-failure rate, retry rate, fallback rate, approval rate, p95 latency, cost per feature. A falling approval rate is a quality signal.

## 11. Evaluation
- Maintain a versioned eval set per feature; grow it from real failures.
- Deterministic scorers first; LLM judges only for free text and only after calibration against human labels.
- Gate prompt/model changes on metrics, with repeat runs to handle nondeterminism.
- Test the orchestration separately with a fake client (see the `testing` skill).

## Checklist before finishing a change
- [ ] Simplest viable mechanism chosen; any step down the ladder justified.
- [ ] Structured output with schema + semantic validation, bounded retries, defined fallback.
- [ ] No model write access; changes go through approved proposals.
- [ ] Untrusted text treated as data; adversarial cases added.
- [ ] Prompt versioned; eval run attached.
- [ ] Failure of the provider degrades gracefully and loses no user input.
- [ ] Run logging and cost limits in place.
