# Language reference

This reference defines how this repository models systems with Stateright.

It is written as an operational guide for building runnable examples, not as a formal grammar.

## File structure

A model project should typically use:

```text
model/
  Cargo.toml
  src/
    lib.rs
    state.rs
    action.rs
    properties.rs
    runner.rs
```

A single-file model is fine for small examples.

### Formatting

- Keep `State`, `Action`, and `Model` in explicit sections.
- Keep property definitions together.
- Keep bounds/config in one model config struct.
- Use short code comments above non-obvious transitions.

### Naming conventions

- `PascalCase`: structs, enums, traits.
- `snake_case`: fields, methods, local variables.
- action enums use clear verbs: `Dispatch`, `Timeout`, `Commit`, `Abort`.
- property names are sentence-like and behavior-oriented.

---

## Model scaffold

### Basic `Model`

```rust
use stateright::{Checker, Model, Property};

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
struct State {
    // bounded state only
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Action {
    // one atomic transition per variant
}

#[derive(Clone)]
struct MyModel {
    // bounds and knobs
}

impl Model for MyModel {
    type State = State;
    type Action = Action;

    fn init_states(&self) -> Vec<Self::State> {
        vec![]
    }

    fn actions(&self, state: &Self::State, actions: &mut Vec<Self::Action>) {
        let _ = state;
        let _ = actions;
    }

    fn next_state(&self, state: &Self::State, action: Self::Action) -> Option<Self::State> {
        let _ = state;
        let _ = action;
        None
    }

    fn properties(&self) -> Vec<Property<Self>> {
        vec![]
    }
}

fn main() {
    let checker = MyModel {}.checker().spawn_bfs().join();
    checker.assert_properties();
}
```

### Actor model

Use `stateright::actor` when behavior is dominated by message scheduling and node-local logic.

---

## State modeling

### Internal state

Keep only behaviorally meaningful information:

- phase/status
- ownership and identity
- counters and deadlines
- in-flight work/messages

Avoid framework and storage details.

### External environment abstraction

Represent external dependencies as abstract effects and bounded environment state.

Example:

- `EmailSent` as a state event marker, not SMTP details.
- `PaymentAuthorized` as an event, not gateway JSON payloads.

### Value shapes

Favor compact, finite representations:

- `u8`/small enums for bounded domains
- capped vectors/maps
- logical ticks instead of wall-clock timestamps

### Enumerations and phase encoding

Prefer enums over boolean combinations.

```rust
#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum InvitationStatus {
    Pending,
    Accepted,
    Declined,
    Expired,
}
```

### Field type guidance

- IDs: small integers where possible
- collections: bounded by model config
- optional values: `Option<T>` only when semantically meaningful

### Relationships in state

Model relationships explicitly with IDs/indexes where needed.

```rust
struct State {
    invitations: BTreeMap<u8, Invitation>,
    candidate_to_invitations: BTreeMap<u8, Vec<u8>>, // bounded
}
```

### Derived values

Keep derived checks as helper methods or property predicates.

```rust
impl State {
    fn pending_count(&self) -> usize {
        self.invitations.values().filter(|i| matches!(i.status, InvitationStatus::Pending)).count()
    }
}
```

---

## Action modeling

### Action structure

Actions represent one atomic decision point.

```rust
#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Action {
    RequestPasswordReset { user_id: u8 },
    CompletePasswordReset { token_id: u8 },
    ExpireResetToken { token_id: u8 },
}
```

### Action generation

`actions(state, actions)` should enumerate candidates, not mutate state.

Use loops to generate bounded options.

### Trigger categories

Common trigger classes:

- external input (`ClientRequest`)
- state transition intent (`PromoteLeader`)
- timer tick/timeout (`Tick`, `LeaseExpired`)
- derived condition check (`QuorumReached`)
- fault injection (`DropMessage`, `DelayMessage`, `CrashNode`)

### Preconditions and guards

Encode guards in `next_state` and return `None` when disabled.

### Local bindings and discard bindings

Use local bindings in `match` arms for clarity. Use `_` when values are irrelevant.

### Transition effects

All state mutation happens in `next_state`.

- clone state
- mutate bounded fields
- return `Some(new_state)`

---

## Properties

### Property structure

Properties are attached in `properties()`:

```rust
fn properties(&self) -> Vec<Property<Self>> {
    vec![
        Property::always("never two leaders in one term", |_m, s| true),
        Property::sometimes("leader is reachable", |_m, s| true),
    ]
}
```

### Safety (`always`)

Use for invariants and forbidden states.

Examples:

- at-most-one-leader
- quota not exceeded
- no duplicate command application

### Reachability (`sometimes`)

Use for witness goals and progress path existence.

### Eventuality (`eventually`)

Use carefully with explicit assumptions. Keep a safety baseline first.

### Property naming

Use names that explain behavior, not implementation details.

Good: `"token cannot be used after expiry"`
Bad: `"if_stmt_line_92_branch_b"`

---

## Checker execution

### BFS

Use for shortest counterexample/witness traces.

```rust
let result = model.checker().spawn_bfs().join();
result.assert_properties();
```

### DFS

Use for lower-memory deeper exploration.

### Simulation

Use when exhaustive exploration is too expensive. Document that it is not proof.

### Explorer UI

```rust
let checker = model.checker().serve("127.0.0.1:3000");
checker.join();
```

Use Explorer to inspect states, outgoing actions, and failing paths.

---

## Config and defaults

### Config

Keep bounds/config in the model struct:

```rust
#[derive(Clone)]
struct ModelCfg {
    max_users: u8,
    max_tokens_per_user: u8,
    token_ttl_ticks: u8,
}
```

### Defaults

`init_states()` defines defaults and should be deterministic and bounded.

---

## Modular modeling

### Namespaces and modules

Split large models into modules (`state`, `action`, `transition`, `properties`) while keeping a single orchestration `impl Model`.

### Using reusable components

Extract repeatable protocol chunks (retry envelopes, dedup logic, lease handling) into helper modules.

### Referencing external behavior

Model external behavior as abstract events and state transitions rather than direct API calls.

### Responding to external triggers

Represent external stimuli as actions and guard conditions.

### Breaking changes

If action/state semantics change, version model fixtures and trace expectations.

### Local examples

Keep runnable examples close to docs where possible.

---

## Validation rules

Before considering a model usable:

1. all state containers are bounded
2. each action has explicit enable/disable semantics
3. each critical invariant is covered by `Property::always`
4. each transition is deterministic for `(state, action)`
5. checker modes are chosen intentionally (BFS/DFS/simulation)
6. assumptions and excluded failures are documented
7. at least one meaningful reachability property exists

---

## Anti-patterns

- unbounded queue/map growth
- hidden randomness in transitions
- wall-clock time in model logic
- properties that only restate transition code trivially
- skipping failure actions in distributed models
- removing properties to silence a failure instead of fixing model/code

---

## Glossary

- **State**: the full modeled system snapshot.
- **Action**: one atomic transition choice.
- **Transition relation**: mapping from `(state, action)` to next state.
- **Safety property**: must hold in all reachable states.
- **Reachability property**: should hold in at least one reachable state.
- **Eventuality property**: should become true along runs under assumptions.
- **Counterexample**: trace that violates a property.
- **Witness**: trace that satisfies a `sometimes` property.
- **Bound**: finite limit applied to state dimensions for tractability.
