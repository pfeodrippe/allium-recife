# Roadmap

This roadmap keeps the original dependency-first style, now focused on Stateright workflows.

## 1. Canonical starter crate

Create a minimal `examples/starter-model/` crate with:

- small `Model` implementation
- safety + reachability properties
- BFS/DFS/Explorer commands
- CI recipe for bounded checks

Purpose: remove setup ambiguity and provide a known-good baseline.

## 2. Counterexample capture and replay

Add guidance + scripts for storing failing traces and replaying them against model revisions.

Purpose: turn one-off checker failures into durable regression artifacts.

## 3. Distillation workbook automation

Extend `skills/distill/references/worked-examples.md` with machine-readable extraction templates:

- state inventory table
- action table
- property candidate table
- abstraction decisions log

Purpose: make code-to-model extraction repeatable across teams.

## 4. Elicitation interview packs by domain

Add domain-specific elicitation packs (auth, consensus, workflow engines, queueing) under `skills/elicit/references/`.

Purpose: accelerate requirement discovery for common distributed patterns.

## 5. Pattern test harnesses

For each pattern in `references/patterns.md`, provide a runnable bounded model and at least one intentionally failing variant.

Purpose: teach by contrast and verify that pattern guidance catches real bugs.

## 6. CI matrix templates

Provide reusable CI templates for:

- quick PR checks (small bounds)
- nightly deep checks
- simulation sweeps for large spaces

Purpose: normalize cost-aware model checking in delivery pipelines.

## 7. Actor-model deep dive

Add a dedicated reference for `actor::ActorModel` covering timer/message nondeterminism and common pitfalls.

Purpose: improve guidance for distributed-node modeling, where most users struggle.
