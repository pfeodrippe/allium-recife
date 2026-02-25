# Tooling roadmap

Quint already gives us parser, typechecker, simulator, tests, and model checking. The remaining opportunity in this repository is workflow tooling: better elicitation, better distillation, and stronger CI integration.

These items are ordered by impact on day-to-day model authoring.

## 1. Project scaffolding for `.qnt` specs

Provide a generator that creates a production-ready Quint layout:

- `model.qnt` with `init`, `step`, and placeholder invariants
- `tests.qnt` with `run` scenarios
- `quint` config defaults for local simulation and CI
- README snippets for `quint parse`, `typecheck`, `run`, `verify`, `test`

This removes blank-page overhead and standardises model shape across teams.

Open questions: scaffold variants per domain (auth/billing/workflow), and whether to generate single-module or multi-module defaults.

## 2. Distillation pipeline from code traces to Quint actions

Distillation currently depends on manual interpretation. Add a guided pipeline that maps implementation events/logs into candidate actions and state updates.

- infer candidate `var` state from persisted entities
- infer candidate actions from endpoints/jobs/consumers
- surface ambiguous transitions for human decisions
- emit a first draft module with TODO markers

Open questions: minimum log schema, confidence scoring for inferred transitions, and how to represent partial certainty.

## 3. Invariant and temporal-property suggestion engine

Given `init` + `step`, suggest checks that teams often miss:

- safety invariants (`count >= 0`, no forbidden states, referential consistency)
- dead-transition checks (actions that are never enabled)
- liveness candidates (eventual completion for pending workflows)

Suggestions should be explicit and reviewable, never auto-accepted.

Open questions: should suggestions be purely static, simulation-assisted, or both.

## 4. MBT trace export workflow

Package a standard flow for model-based testing using Quint traces:

- `quint run --mbt --out-itf=...`
- adapters for common test harnesses
- replay support in CI so counterexamples become regression tests

This turns model traces into executable implementation checks.

Open questions: canonical trace schema per target language and trace-to-test mapping conventions.

## 5. Continuous drift detection (spec vs implementation)

Add a CI-focused drift checker that combines static checks and trace checks:

- compare implementation behaviour against expected action traces
- report mismatches using model terminology (action + state predicate)
- classify findings: spec bug, code bug, or intentional divergence

Open questions: balancing signal/noise, handling environment nondeterminism, and integration with existing observability stacks.
