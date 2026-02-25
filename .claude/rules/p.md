---
globs: "**/*.p"
---

# P language

P models asynchronous distributed behavior using communicating state machines and observer monitors.

## File focus

- `.p`: events, types, machines, specs, modules, tests
- `.pproj`: project input/output wiring for `p compile`

## Semantics reminders

- `send` is asynchronous; machines communicate via queued events.
- Machines have local state; do not model hidden shared memory implicitly.
- `spec` monitors are observer-only and cannot cause side effects.
- Liveness should use `hot` states, not prose comments.

## Common syntax pitfalls

- Missing or duplicated `start state` in a machine.
- Using direct call style where event protocol is required.
- Forgetting correlation IDs in request/response pairs.
- Overusing `ignore` and accidentally dropping required events.

## Modeling rules

- Keep machine responsibilities narrow.
- Put reusable logic behind module boundaries.
- Put global properties in `spec` monitors.
- Add testcases for nominal, failure, timeout, and scale scenarios.

## Tooling

- Compile with `p compile` (prefer `.pproj`).
- Check with `p check -tc <name> -s <count>`.
