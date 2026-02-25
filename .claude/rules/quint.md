---
globs: "**/*.qnt"
---

# Quint language

Quint is an executable specification language for modeling state machines with explicit transitions and checkable properties.

## File structure

Use this order unless there is a strong reason not to:

1. `import`
2. `type`
3. `const` + `assume`
4. `var`
5. helpers (`pure def`, `val`)
6. `action init`
7. domain actions
8. `action step`
9. invariants (`val`) and temporal properties (`temporal`)
10. `run` scenarios

## Syntax pitfalls

**Primed state (`x'`)**
- Use primed vars only inside actions.
- `x` is current state, `x'` is next state.

**`all` vs `any`**
- `all { ... }` is conjunction (guards + assignments that all apply).
- `any { ... }` is non-deterministic branch choice among alternatives.

**Non-determinism**
- Prefer explicit `nondet v = oneOf(...)` bindings with finite domains.

**State preservation**
- If an action should not change a variable, assign it explicitly (`x' = x`).

**Finite modeling for checks**
- Keep domains finite (`Set(...)`, bounded ranges) for practical `run`/`verify`.

## Anti-patterns

- Encoding API/ORM/infrastructure details in domain state.
- Monolithic `step` actions with hidden behavior branches.
- Missing time guards for expiry/reset behavior.
- Unnamed critical invariants.
- No failure-path scenarios in `run` definitions.

## Naming conventions

- Modules and sum constructors: `PascalCase`
- Actions, helpers, invariants: `camelCase`
- Constants: `UPPER_SNAKE_CASE` (when config-like)

## Reference

See `references/language-reference.md` for the repository's Quint modeling conventions.
