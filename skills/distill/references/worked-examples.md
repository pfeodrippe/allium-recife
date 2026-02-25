# Worked examples: code to Stateright model

These examples show how to distill behavior from implementation code into bounded Stateright models.

## Example 1: Idempotent command apply

### Implementation sketch (Rust service)

```rust
pub fn apply_delta(store: &mut Store, req_id: u64, delta: i64) -> i64 {
    if let Some(cached) = store.responses.get(&req_id) {
        return *cached;
    }

    store.value += delta;
    let result = store.value;
    store.responses.insert(req_id, result);
    result
}
```

### Distillation decisions

Keep:

- dedup by `req_id`
- exactly-once side effect on `value`
- stable cached response

Drop:

- storage backend details
- API transport details

### Distilled model

```rust
use std::collections::BTreeMap;
use stateright::{Model, Property};

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
struct State {
    value: i8,
    responses: BTreeMap<u8, i8>,
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Action {
    Submit { req_id: u8, delta: i8 },
}

#[derive(Clone)]
struct IdempotencyModel;

impl Model for IdempotencyModel {
    type State = State;
    type Action = Action;

    fn init_states(&self) -> Vec<Self::State> {
        vec![State { value: 0, responses: BTreeMap::new() }]
    }

    fn actions(&self, _state: &Self::State, actions: &mut Vec<Self::Action>) {
        for req_id in 0..=2 {
            for delta in [-1, 1] {
                actions.push(Action::Submit { req_id, delta });
            }
        }
    }

    fn next_state(&self, state: &Self::State, action: Self::Action) -> Option<Self::State> {
        match action {
            Action::Submit { req_id, delta } => {
                let mut s = state.clone();
                if s.responses.contains_key(&req_id) {
                    return Some(s);
                }
                s.value = s.value.saturating_add(delta);
                s.responses.insert(req_id, s.value);
                Some(s)
            }
        }
    }

    fn properties(&self) -> Vec<Property<Self>> {
        vec![
            Property::always("responses never shrink", |_m, s| s.responses.len() <= 3),
            Property::sometimes("value changes are reachable", |_m, s| s.value != 0),
        ]
    }
}
```

### Notes

This first model intentionally under-checks invariants. In practice, add a stronger safety property asserting replay of the same `req_id` does not mutate `value`.

---

## Example 2: Retry worker with terminal failure

### Implementation sketch (TypeScript worker)

```ts
if (job.status === "pending") {
  job.status = "in_flight";
}

if (result === "transient_error") {
  job.attempts += 1;
  if (job.attempts >= job.maxAttempts) {
    job.status = "failed_terminal";
  } else {
    job.status = "pending";
  }
}

if (result === "success") {
  job.status = "succeeded";
}
```

### Distillation decisions

Keep:

- state machine (`pending -> in_flight -> pending/succeeded/failed_terminal`)
- bounded attempts

Drop:

- queue implementation
- transport protocol
- logging/telemetry

### Distilled model

```rust
use stateright::{Model, Property};

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Status { Pending, InFlight, Succeeded, FailedTerminal }

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
struct State {
    attempts: u8,
    max_attempts: u8,
    status: Status,
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Action {
    Dispatch,
    TransientError,
    Success,
}

#[derive(Clone)]
struct RetryModel;

impl Model for RetryModel {
    type State = State;
    type Action = Action;

    fn init_states(&self) -> Vec<Self::State> {
        vec![State { attempts: 0, max_attempts: 2, status: Status::Pending }]
    }

    fn actions(&self, _state: &Self::State, actions: &mut Vec<Self::Action>) {
        actions.push(Action::Dispatch);
        actions.push(Action::TransientError);
        actions.push(Action::Success);
    }

    fn next_state(&self, state: &Self::State, action: Self::Action) -> Option<Self::State> {
        let mut s = state.clone();
        match action {
            Action::Dispatch if matches!(s.status, Status::Pending) => {
                s.status = Status::InFlight;
                Some(s)
            }
            Action::TransientError if matches!(s.status, Status::InFlight) => {
                s.attempts += 1;
                s.status = if s.attempts >= s.max_attempts {
                    Status::FailedTerminal
                } else {
                    Status::Pending
                };
                Some(s)
            }
            Action::Success if matches!(s.status, Status::InFlight) => {
                s.status = Status::Succeeded;
                Some(s)
            }
            _ => None,
        }
    }

    fn properties(&self) -> Vec<Property<Self>> {
        vec![
            Property::always("attempts bounded", |_m, s| s.attempts <= s.max_attempts),
            Property::always("terminal states are terminal", |_m, s| {
                !matches!(s.status, Status::Succeeded | Status::FailedTerminal) || s.attempts <= s.max_attempts
            }),
            Property::sometimes("success reachable", |_m, s| matches!(s.status, Status::Succeeded)),
        ]
    }
}
```

### Notes

If production has duplicate result delivery, add an explicit duplicate action and safety property proving no illegal state transition from terminal states.

---

## Distillation checklist used in both examples

1. identify meaningful state
2. define atomic actions
3. encode guards as `None` transitions
4. bound identifiers/counters
5. encode at least two safety properties
6. run BFS first for short traces

---

## Example 3: Soft delete lifecycle (Java/Spring style service)

### Implementation sketch

```java
if (doc.status == ACTIVE) {
    doc.status = DELETED;
    doc.deletedAt = now();
}

if (restoreRequested && doc.status == DELETED) {
    doc.status = ACTIVE;
    doc.deletedAt = null;
}
```

### Distillation decisions

Keep:

- explicit lifecycle states
- delete/restore guard conditions
- visibility semantics for list/read behavior

Drop:

- persistence annotations
- controller mapping details
- timestamp precision details

### Distilled model

```rust
use stateright::{Model, Property};

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Status {
    Active,
    Deleted,
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
struct State {
    status: Status,
    visible_in_list: bool,
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum Action {
    Delete,
    Restore,
    RefreshProjection,
}

#[derive(Clone)]
struct SoftDeleteModel;

impl Model for SoftDeleteModel {
    type State = State;
    type Action = Action;

    fn init_states(&self) -> Vec<Self::State> {
        vec![State {
            status: Status::Active,
            visible_in_list: true,
        }]
    }

    fn actions(&self, _state: &Self::State, actions: &mut Vec<Self::Action>) {
        actions.push(Action::Delete);
        actions.push(Action::Restore);
        actions.push(Action::RefreshProjection);
    }

    fn next_state(&self, state: &Self::State, action: Self::Action) -> Option<Self::State> {
        let mut s = state.clone();
        match action {
            Action::Delete if matches!(s.status, Status::Active) => {
                s.status = Status::Deleted;
                Some(s)
            }
            Action::Restore if matches!(s.status, Status::Deleted) => {
                s.status = Status::Active;
                Some(s)
            }
            Action::RefreshProjection => {
                s.visible_in_list = matches!(s.status, Status::Active);
                Some(s)
            }
            _ => None,
        }
    }

    fn properties(&self) -> Vec<Property<Self>> {
        vec![
            Property::always("deleted docs are not visible", |_m, s| {
                !matches!(s.status, Status::Deleted) || !s.visible_in_list
            }),
            Property::sometimes("delete then restore is reachable", |_m, s| {
                matches!(s.status, Status::Active)
            }),
        ]
    }
}
```
