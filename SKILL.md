---
name: stateright
description: Rust model checking for distributed systems. Use this skill when writing, reviewing, or debugging Stateright models and properties.
version: 1
auto_trigger:
  - file_patterns: ["**/*.rs"]
  - keywords: ["stateright", "model checker", "Property::always", "ActorModel", "counterexample"]
---

# Stateright

Stateright is a Rust framework for executable modeling and model checking. It is most useful where concurrency, failures, and interleavings matter.

This skill focuses on building and maintaining runnable models, then using checker output to guide implementation changes.

## Routing table

| Task | Skill | When |
|------|-------|------|
| Writing or debugging Stateright Rust models | this skill | You need trait/API guidance and modeling patterns |
| Building a model from stakeholder intent | `elicit` | Requirements are conversational or ambiguous |
| Extracting a model from existing code | `distill` | Implementation exists and you need a faithful model |

## Quick modeling summary

### State

Model state should be:

- finite
- hashable/equatable (`Clone + Eq + Hash`)
- behaviorally meaningful (not implementation plumbing)

Use compact enums and bounded collections.

### External environment

Represent external systems as abstract effects and bounded state, not concrete APIs. Example: model `EmailQueued` or `PaymentFailed` as events/state transitions, not provider payload formats.

### Value domains and enums

Prefer small explicit domains over free-form strings. A phase enum is usually better than multiple booleans.

### Model configuration

Put bounds and knobs in model config structs (`max_nodes`, `max_inflight`, `max_retries`, tick horizon). Keep them explicit and versioned.

### Transition relation

`next_state(state, action) -> Option<State>` is the core semantic contract:

- `Some` => enabled and applied
- `None` => disabled in this state

Transitions must be deterministic for `(state, action)`.

### Trigger/action types

Common action families:

- external input (`ClientRequest`)
- protocol steps (`GrantVote`, `CommitEntry`)
- timer behavior (`Tick`, `LeaseTimeout`)
- fault behavior (`DropMessage`, `CrashNode`, `DelayDelivery`)

### Action-level iteration

`actions(state, actions)` enumerates enabled candidates from bounded domains. It should not mutate state.

### Property patterns

- `Property::always`: safety invariants and forbidden states
- `Property::sometimes`: witness/reachability goals
- `Property::eventually`: eventuality claims with explicit assumptions

Start with safety, then reachability.

### Actor model

Use `actor::ActorModel` when node-local behavior and message scheduling are dominant concerns. Keep mailbox/timer nondeterminism explicit and bounded.

### Model-to-implementation contract

Treat models and code as complementary artifacts:

- model says what must hold across schedules
- code says how runtime behavior is produced

Drift between them should be classified (model bug, code bug, intentional abstraction, aspirational behavior).

### Predicate expressions

Property predicates should be stable and readable. Prefer helper methods on `State` for repeated logic.

### Modular model layout

For larger models split into modules:

- `state`
- `action`
- `transition`
- `properties`
- `runner`

### Bounds

Bounds are first-class. Tighten first for speed, widen intentionally for deeper checks.

### Defaults/init

`init_states()` should be deterministic and minimal. Multiple initial states are allowed when uncertainty in start conditions is meaningful.

### Deferred assumptions

Document what is intentionally excluded (fair delivery, no crash recovery, bounded retries). Deferred assumptions should be visible near model config/properties.

### Open questions

When requirements are unclear, record explicit model questions instead of guessing fairness, retries, or ownership semantics.

## Checker execution patterns

Use a checker from `.checker()` and choose strategy by goal:

- `spawn_bfs()` for shortest counterexample/witness traces.
- `spawn_dfs()` for deeper low-memory exploration.
- `spawn_simulation(...)` for large spaces where exhaustive checking is not tractable.
- `serve(addr)` for interactive Explorer debugging.

Typical assertion flow:

```rust
let checker = model.checker().spawn_bfs().join();
checker.assert_properties();
```

## Design rules

1. Bound the state space aggressively.
2. Encode one atomic semantic step per action.
3. Keep transitions pure and deterministic.
4. Prefer phase enums to boolean combinations.
5. Add 2-5 core safety properties before liveness work.
6. Keep failure semantics explicit where they matter.
7. Add symmetry/reduction only after baseline correctness is established.

## Anti-patterns

- unbounded collections in state
- hidden nondeterminism (`rand`, wall-clock time, I/O)
- implementation-heavy state fields with no behavioral impact
- giant action enums with weak guards
- liveness claims with no safety baseline
- mutating state in `actions()`

## Counterexample workflow

When a property fails:

1. re-run with BFS for shortest trace
2. replay trace step-by-step in model code
3. classify (model bug, code bug, missing/incorrect property)
4. patch model/properties first
5. preserve trace shape as regression artifact

## When to use this skill

Use this skill when the user asks to:

- add or fix Stateright models
- define properties/invariants
- investigate counterexamples
- model distributed behavior in Rust
- set up model-checking CI

## References

- [Stateright docs](https://docs.rs/stateright)
- [Project site](https://www.stateright.rs/)
- [Repo and examples](https://github.com/stateright/stateright)
- [Local language reference](./references/language-reference.md)
- [Local pattern catalog](./references/patterns.md)
- [Local test-generation guidance](./references/test-generation.md)
