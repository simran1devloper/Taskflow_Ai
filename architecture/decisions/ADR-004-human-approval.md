# ADR-004: Human Approval for AI-Initiated Changes

- **Status:** Proposed (awaiting approval)
- **Date:** 2026-10-30

## Context

LLM output can be wrong, malformed, or manipulated (for example by prompt injection through task text or imported content). Task data belongs to the user, and silent AI edits erode trust.

## Open question (required before acceptance)

What should the user see when AI parsing fails: a silently created plain task, a retry prompt, or the raw text for manual edit? This likely differs between quick capture (never lose the input) and AI planning (explicit error).

## Decision (provisional)

- AI output that would change data is a **proposal**, not a write. The user reviews it and approves, edits, or rejects it before anything is committed.
- On approval, deterministic code applies the proposal in **one DB transaction** together with an **audit log** entry (who, what, proposal ID, model and prompt version).
- The LLM never has direct write access to the database.
- Read-only AI actions (search, summarize, answer) need no approval.
- Proposals are validated before being shown (schema plus semantic checks) and re-validated at apply time, since the underlying data may have changed. Stale proposals are rejected.
- Proposals have an expiry and a status (`pending`, `approved`, `rejected`, `expired`, `failed`).
- Possible later relaxation: auto-apply for low-risk, easily reversible actions, only if enabled by the user and backed by an undo mechanism and eval data.

## Alternatives considered

- **Auto-apply all AI changes:** rejected. Wrong or injected output would directly corrupt user data.
- **Approval only for destructive actions:** rejected for the MVP. Incorrect creates and edits are also user-visible errors. This can be revisited with evidence.

## Consequences

- A safety net behind every other validation layer, and a clear audit trail.
- Extra UI work (proposal review) and one extra table for proposals.
- Slightly more friction for the user; mitigated by making review fast (diff view, one-click approve).
