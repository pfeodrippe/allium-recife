# Test generation and CI from Stateright models

Model checking and tests should reinforce each other.

## 1. Generate test ideas from properties

For each property, derive three concrete test families:

- **Positive witness tests**: confirm intended valid behavior paths.
- **Negative regression tests**: replay known failing traces and assert they stay fixed.
- **Boundary tests**: max/min bounds that stress guards and transitions.

## 2. Derive tests from counterexamples

When Stateright finds a counterexample:

1. save trace actions as fixture data
2. replay trace in unit/integration test harness
3. assert the previous violation no longer occurs

This keeps checker discoveries alive in the implementation suite.

## 3. Transition-level tests

For each action variant:

- enabled-case test (`next_state` returns `Some`)
- disabled-case test (`next_state` returns `None`)
- idempotency/replay checks where relevant

## 4. Invariant-aligned tests

Every `Property::always` should map to at least one runtime assertion in code tests.

Examples:

- "at most one leader" -> cluster-state assertion in simulation/integration tests
- "attempts <= max_attempts" -> retry manager unit test
- "dedup prevents double apply" -> API retry integration test

## 5. CI tiers

- **PR tier**
  - small bounds
  - BFS
  - strict runtime budget

- **Nightly tier**
  - deeper bounds
  - DFS or simulation sweeps
  - artifact upload for traces

- **Release tier**
  - targeted deep checks on critical protocols
  - explicit sign-off for any skipped property

## 6. Minimal CI checklist

- model checks run on every merge request
- failing traces are archived
- runtime tests include at least one replayed model counterexample
- bounds/config used in CI are versioned
- property names are stable and meaningful

## 7. What not to do

- Do not run only `sometimes` properties in CI.
- Do not treat simulation runs as exhaustive proof.
- Do not change bounds silently when checks become slow.
