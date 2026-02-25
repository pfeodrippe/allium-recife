# Stateright

*Velocity through executable confidence*

---

A skill pack for designing, checking and evolving Rust models with [Stateright](https://www.stateright.rs/).

This repository keeps the same operational structure (`SKILL.md`, `skills/elicit`, `skills/distill`, `references`, `.claude` agents/rules), now oriented around Stateright model checking.

## Get started

If your assistant supports local skills, point it at this repository and invoke `/stateright`.

Main entry points:

- `/stateright` - build or refine a Stateright model
- `/stateright:elicit` - turn requirements into model state/actions/properties
- `/stateright:distill` - reverse engineer a model from existing Rust code

## Working with agents

A rule in [.claude/rules/stateright.md](.claude/rules/stateright.md) loads when editing Rust files so routine model work does not require re-explaining Stateright semantics.

Two delegated agents mirror the original project structure:

- [tend](.claude/agents/tend.md) - grows and refactors Stateright models
- [weed](.claude/agents/weed.md) - checks divergence between model and implementation

## The problem with conversational context

- Within a session, details drift and hidden assumptions accumulate.
- Across sessions, critical concurrency assumptions are lost.

Stateright models keep those assumptions executable and stable.

## Why not just point the LLM at the code?

Code shows one implementation path. It does not exhaustively show all interleavings and failure schedules that can occur in production.

Stateright explores schedules and state combinations your tests and prompts usually miss.

## Why not capture this in markdown?

Markdown is useful for discussion, but it does not execute transitions or check invariants. Stateright models do both.

## Iterating on models and implementation

The workflow is cyclical:

1. capture intent as a bounded model
2. check properties and inspect traces
3. update implementation
4. distill behavior back into the model

Elicitation and distillation are both first-class in this repository to support that loop.

## On single sources of truth

The model and implementation are intentionally separate artifacts with different jobs:

- model: what must hold under all explored schedules
- implementation: how the runtime system is built

Disagreement between them is signal, not noise.

## Why Stateright

Prompt-only implementation work fails under concurrency pressure. Stateright gives the team:

- executable models in Rust (`Model` / `actor::ActorModel`)
- explicit nondeterminism (`actions` + `next_state`)
- checkable properties (`Property::always`, `Property::sometimes`, `Property::eventually`)
- reproducible traces for counterexamples and witness paths
- an Explorer UI for path/state inspection

## What this looks like

A minimal model:

```rust
use stateright::{Checker, Model, Property};

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
struct CounterState {
    value: u8,
}

#[derive(Clone, Debug, Eq, Hash, PartialEq)]
enum CounterAction {
    Inc,
    Dec,
}

#[derive(Clone)]
struct CounterModel {
    max: u8,
}

impl Model for CounterModel {
    type State = CounterState;
    type Action = CounterAction;

    fn init_states(&self) -> Vec<Self::State> {
        vec![CounterState { value: 0 }]
    }

    fn actions(&self, _state: &Self::State, actions: &mut Vec<Self::Action>) {
        actions.push(CounterAction::Inc);
        actions.push(CounterAction::Dec);
    }

    fn next_state(&self, last_state: &Self::State, action: Self::Action) -> Option<Self::State> {
        match action {
            CounterAction::Inc if last_state.value < self.max => {
                Some(CounterState { value: last_state.value + 1 })
            }
            CounterAction::Dec if last_state.value > 0 => {
                Some(CounterState { value: last_state.value - 1 })
            }
            _ => None,
        }
    }

    fn properties(&self) -> Vec<Property<Self>> {
        vec![
            Property::always("value never exceeds max", |m, s| s.value <= m.max),
            Property::sometimes("max is reachable", |m, s| s.value == m.max),
        ]
    }
}

fn main() {
    let checker = CounterModel { max: 3 }
        .checker()
        .spawn_bfs()
        .join();
    checker.assert_properties();
}
```

Explorer usage:

```rust
let checker = CounterModel { max: 3 }
    .checker()
    .serve("127.0.0.1:3000");
checker.join();
```

## Distillation and elicitation

This repository supports both directions of model work:

- **Elicitation** (`skills/elicit`) starts from intent and captures state, actions, boundaries and properties.
- **Distillation** (`skills/distill`) starts from existing code/traffic/tests and extracts a minimal, checkable model.

Both workflows converge to the same deliverable: a bounded model that catches real failures before implementation drift reaches production.

## Repository map

- [SKILL.md](SKILL.md) - primary Stateright skill
- [references/language-reference.md](references/language-reference.md) - trait/API-oriented reference
- [references/patterns.md](references/patterns.md) - reusable modeling patterns
- [references/test-generation.md](references/test-generation.md) - test and CI guidance
- [skills/elicit/SKILL.md](skills/elicit/SKILL.md) - requirement-to-model workflow
- [skills/distill/SKILL.md](skills/distill/SKILL.md) - code-to-model workflow

## Governance docs

The existing governance structure remains, but now evaluates modeling quality and checker ergonomics:

- [TEAM.md](TEAM.md) - review panel roles
- [REVIEW.md](REVIEW.md) - rough-edge review prompt
- [PROPOSE.md](PROPOSE.md) - feature/approach proposal prompt
- [TODO.md](TODO.md) - roadmap

## External references

- Stateright site: <https://www.stateright.rs/>
- API docs: <https://docs.rs/stateright>
- Source/examples: <https://github.com/stateright/stateright>

## Feedback

If a model pattern or skill workflow is missing, open an issue or submit a PR with a counterexample-driven use case.

## License

MIT. See [LICENSE](LICENSE).
