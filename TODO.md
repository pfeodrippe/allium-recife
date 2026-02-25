# Tooling roadmap

This repository is P-first. The roadmap keeps the original tooling intent while targeting `.p` and `.pproj` artifacts.

## 1. Parser and structural validator

Add deterministic validation over P project conventions:

- missing/duplicate `start state`
- unresolved event/type/function references
- monitor misuse of forbidden operations
- test modules missing intended monitor assertions

## 2. Property-based test generation

Generate testcase scaffolding from machines and monitors:

- nominal success paths
- per-guard rejection paths
- timeout/retry paths
- parameterized scale paths

## 3. Runtime trace validation

Define trace schemas aligned to modeled P events and monitor expectations.

Goal: compare production traces to model contracts and report drift in model terms.

## 4. Model checking bridge

Provide checker profile automation:

- `smoke`: `-s 1`
- `ci`: `-s 100`
- `deep`: `-s 10000+`

Longer-term: derive schedule budgets from testcase classes and model complexity.

## 5. Formal guarantee integration

Map human guarantees to executable evidence:

- guarantee-to-monitor linkage
- coverage tracking for critical properties
- stale guarantee detection after model changes
