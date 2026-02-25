---
name: distill
description: Use this skill when the user has existing code and wants to extract a bounded Stateright model and property suite from real behavior.
---

# Distillation guide

This guide covers extracting Stateright models from existing implementation code.

The core task is separating behavioral semantics from implementation mechanics while preserving concurrency and failure behavior.

## Scoping the distillation effort

Define scope before reading every module.

### Questions to ask first

1. Which subsystem is being modeled?
2. Which incidents or regressions should the model capture?
3. Which layers are excluded (framework, persistence, telemetry)?
4. What time budget/bounds are acceptable for first checker runs?

### The "Would we rebuild this?" test

Ask for each behavior: "If we rebuilt today, would this still be required behavior?"

- yes -> candidate for model
- no -> legacy/mechanical noise, usually exclude

### Documenting scope decisions

Record:

```text
Scope: ...
Includes: ...
Excludes: ...
Failure model: ...
Bounds: ...
```

## Finding the right level of abstraction

### The "Why" test

Why does this detail matter for correctness under interleavings?

Include details that affect:

- allowed/disallowed transitions
- ownership/exclusivity
- idempotency/duplication
- ordering/timeouts/retries

### The "Could it be different?" test

If the implementation detail can change without changing required behavior, abstract it.

### The "Template vs Instance" test

Model protocol classes first:

- template: retry envelope, lease, quorum, dedup
- instance: specific queue vendor, ORM API, HTTP framework

### Abstraction sanity checks

- are state fields finite and behaviorally meaningful?
- can each action be explained in one semantic sentence?
- does each property guard a real risk?

## The distillation mindset

### Code is over-specified

Production code includes storage, transport, retries, and integration glue that may not all belong in model state.

### Ask "Would a model checker care?"

If a detail does not influence reachable states or properties, do not model it.

### Distinguish means from ends

- means: thread pools, ORM transactions, API wrappers
- ends: ordering constraints, uniqueness guarantees, state machine transitions

## The concrete detail problem

### Vendor/protocol instance example

If code references a specific broker/library, ask whether behavior is generic (e.g., at-least-once delivery) or vendor-specific and product-critical.

### Database choice example

SQL/NoSQL specifics are usually implementation details. Model resulting behavior (e.g., read-after-write visibility assumptions) instead.

### Third-party integration example

Include integration detail only when product behavior depends on it directly; otherwise model as abstract external event/effect.

### The "Multiple implementations" heuristic

If multiple implementations exist in code, the variation may be domain-level and worth modeling explicitly.

## Distillation process

### Step 1: Map the territory

Identify:

- entry points (handlers/consumers/jobs)
- state holders
- transition logic
- external boundaries

### Step 2: Extract entity/state phases

Find implicit and explicit phase/status machines in code and make them explicit in model enums.

### Step 3: Extract transitions

For each important flow path:

- preconditions
- atomic transition steps
- resulting state changes

### Step 4: Find temporal/failure triggers

Look for timers/retries/timeouts and fault behavior:

- retry loop triggers
- lease/lock expiry
- periodic cleanup windows
- drop/reorder/duplicate handling

### Step 5: Identify external boundaries

Represent external inputs/outputs as actions or bounded environment state, not raw API traffic.

### Step 6: Abstract away implementation

Remove framework/storage artifacts while preserving transition semantics.

### Step 7: Validate with stakeholders

Walk one happy path and one failure path against both model and code owners.

## Recognizing reusable model component candidates

### Signals in the code

- repeated retry envelope logic
- repeated lease ownership logic
- repeated dedup/idempotency maps
- repeated quorum bookkeeping

### Questions to ask

1. Is this protocol logic reused across services?
2. Are the same invariants expected everywhere?
3. Would a shared model component reduce divergence risk?

### How to handle

- reuse existing pattern
- extract local shared model module
- inline only when truly domain-specific

### Red flags: integration logic in your model core

If model code is dominated by vendor/client library behavior, abstraction boundary is likely wrong.

### Common reusable extractions

- retry/backoff envelope
- lease manager
- dedup cache
- quorum helper

## Common distillation challenges

### Challenge: Duplicate terminology

Normalize terms before writing actions/properties. One concept should have one stable name.

### Challenge: Implicit state machines

Many systems encode phase via booleans or scattered checks. Convert to explicit enums for model clarity.

### Challenge: Scattered logic

Behavior often spans handler + service + store. Distill across files, not per-file.

### Challenge: Dead code and historical accidents

Do not model dead branches unless they are still behaviorally reachable in production.

### Challenge: Missing error handling

Absence in code may itself be critical. Capture as model assumption or explicit open gap.

### Challenge: Over-engineered abstractions

Collapse wrappers and adapter layers into simple action/state semantics.

## Checklist: Have you abstracted enough?

- state fields are finite and small
- no framework plumbing in state
- transitions are atomic and readable
- properties reflect behavior, not line-by-line code

## Checklist: Terminology consistency

- one term per concept
- action names align with domain language
- property names are behavior-oriented and stable

## After distillation

Before signing off:

1. run BFS with small bounds
2. inspect failures/witnesses
3. classify divergences (model bug, code bug, intentional abstraction)
4. record assumptions and exclusions
5. plan next bound expansion

## References

- [worked examples](./references/worked-examples.md)
- [language reference](../../references/language-reference.md)
- [pattern catalog](../../references/patterns.md)
