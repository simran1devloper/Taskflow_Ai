---
name: frontend-development
description: Conventions for building React + TypeScript UI - feature structure, server vs local state, generated API types, forms, error/loading states, accessibility, and testing. Use when adding or changing any frontend screen, component, hook, or API integration.
---

# Frontend Development

Reusable rules for UI code. Screen-level requirements live in `plans/`; this file is about *how* to build them.

## Structure
```
src/
  features/<name>/   # components, hooks, api calls, tests for one area
  shared/            # ui primitives, utilities used by 2+ features
  api/               # generated client, fetch wrapper
```
- Organize by feature, not by file type. A feature may import from `shared`, never from another feature's internals; share through `shared` or a feature's public `index.ts`.
- Promote code to `shared` only on its second real use.

## State
| Kind | Where |
|---|---|
| Server data (lists, entities) | TanStack Query. Never copy it into local state. |
| Form input | React Hook Form |
| Ephemeral UI (open/closed, selected tab) | `useState` near where it is used |
| URL-representable (filters, selected item) | Router search params, so links and refresh work |

- Do not add a global store until there is state that none of the above fits.
- Derive values during render instead of syncing them with effects. Avoid `useEffect` for data that can be computed.

## API integration
- Types come from the generated client built from the backend OpenAPI schema. Never hand-write request or response types; a backend change should break the build.
- One fetch wrapper handles credentials, CSRF header, error-envelope parsing, and 401 handling. Components never call `fetch` directly.
- Mutations: invalidate the specific query keys they affect. Use optimistic updates only when rollback is simple and tested.
- Send an idempotency key on create/apply actions and disable the control while the request is pending.

## Forms and validation
- Validate with a Zod schema that mirrors server rules, but treat the server as the authority and display its field errors.
- Never lose user input on failure: keep the form populated and show the error next to it.

## Every async view has four states
Loading, empty, error, success. Error states show a human message and a way to retry; include the request ID in a "copy details" action.

## Accessibility (baseline, not optional)
- Use semantic elements (`button`, `a`, `label`, `nav`). Do not attach click handlers to `div`.
- Every input has a label; every icon-only button has an accessible name.
- Everything works by keyboard; focus is visible and managed in dialogs.
- Do not convey meaning by color alone.

## Security
- Never render untrusted content with `dangerouslySetInnerHTML`. Render AI output and user text as text.
- No secrets or tokens in client code or `localStorage`. Auth is cookie-based.
- Treat AI-generated content as untrusted: show it clearly as a suggestion and require explicit user action to apply it.

## Performance
- Measure before optimizing. Memoize only for a demonstrated problem.
- Paginate or virtualize long lists; debounce search inputs.
- Code-split by route.

## Testing
- Component tests assert user-visible behavior (roles, text), not implementation details.
- Mock the network with MSW at the HTTP layer, not by mocking hooks.
- Cover a few critical flows end to end; do not duplicate component tests there.
- Run axe checks on main screens.

## Checklist before finishing a change
- [ ] No hand-written API types.
- [ ] Loading, empty, error and success states exist.
- [ ] Keyboard and screen-reader usable.
- [ ] User input survives failure.
- [ ] Types strict, lint clean, tests added.
