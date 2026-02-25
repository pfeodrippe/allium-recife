---
globs: "**/*.rs"
---

# Stateright modeling rule

Treat Rust files in this workspace as potential model files when the context indicates model checking.

## Core expectations

- Use explicit `State` and `Action` representations with bounded domains.
- Keep `actions` side-effect free; mutate only in `next_state`.
- Return `None` for disabled transitions.
- Prefer `Property::always` for core safety guarantees.
- Add `Property::sometimes` for meaningful reachability checks.
- Use `Property::eventually` only with explicit assumptions.

## Modeling pitfalls to avoid

- unbounded collections in state
- hidden randomness or time in transitions
- implementation details that do not affect behavior
- giant action enums with weak guards

## Checker usage defaults

- start with `spawn_bfs()` to get short traces
- use `spawn_dfs()` for deeper memory-efficient checks
- use `serve(...)` to inspect traces interactively

## Reference

See `references/language-reference.md` for this repository's Stateright guidance.
