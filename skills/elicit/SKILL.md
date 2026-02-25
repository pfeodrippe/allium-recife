---
name: elicit
description: Use this skill when the user wants to build a Stateright model from requirements, elicit behavior under concurrency, and define checkable properties before coding.
---

# Elicitation

This skill guides requirement conversations toward a runnable Stateright model.

The target artifact is executable Rust model code with bounded state, explicit actions, and properties. The target is not prose requirements alone.

## Scoping the model

Before discussing transitions, define the boundary.

### Questions to ask first

1. What subsystem are we modeling right now?
2. What is in scope vs deliberately out of scope?
3. Which failures/interleavings matter most?
4. What level of fidelity is expected for v1 of the model?
5. Which existing production incidents should this model be able to explain?

### Documenting scope decisions

Capture this at the top of the session output:

```text
Model scope: ...
Includes: ...
Excludes: ...
Failure model: drop/delay/reorder/crash? ...
Bounds target: ...
```

If scope is broad, split into multiple small models instead of one huge first pass.

## Finding the right level of abstraction

The core elicitation judgment is deciding what belongs in modeled state and what should remain implementation detail.

### The "Why" test

Ask of each detail: "Why would a correctness discussion care about this?"

- If it affects guards, ownership, ordering, uniqueness, or invariants: include it.
- If it is storage/wiring/framework detail: abstract it.

### The "Could it be different?" test

Ask: "Could implementation change while required behavior stays the same?"

- If yes, likely implementation detail.
- If no, likely domain/protocol behavior and should be modeled.

### The "Template vs Instance" test

Prefer behavior categories over vendor instances unless the instance is itself product behavior.

- Template: "message delivery can fail"
- Instance: "Kafka ack mode X"

Model the template first.

### Levels of abstraction

```text
Too abstract:   "system remains reliable"
Useful level:   "no command is applied twice"
Too concrete:   "method foo() line 87 updates map in transaction"
```

### Configuration vs hardcoding

If values may vary by environment or policy, place them in model config (`max_retries`, `lease_ticks`) rather than hardcoded constants.

### Black boxes

Keep algorithms as black boxes when their internals are not the current question.

Example: model "scheduler chooses one ready task" without modeling full heuristic internals unless fairness/performance semantics are in scope.

## Elicitation methodology

### Phase 1: Scope definition

Goal: choose model boundary and success criteria.

Outputs:

- in-scope behaviors
- excluded behaviors
- failure assumptions
- initial bounds

### Phase 2: Happy path flow

Goal: extract nominal transition structure before edge cases.

Ask for one full story end-to-end and convert to:

- initial state
- action sequence
- resulting state changes

### Phase 3: Edge cases and errors

Goal: expose race conditions and failure transitions.

Probe for:

- retries and duplicates
- timeouts
- out-of-order delivery
- stale reads
- conflicting concurrent actions

Each edge case should become explicit action(s) and guard(s), not just notes.

### Phase 4: Refinement

Goal: stabilize model shape and properties.

Checks:

- are states finite and bounded?
- are actions atomic?
- do properties reflect real risks?
- are assumptions visible?

## Elicitation principles

### Ask one question at a time

Single high-impact questions produce clearer, testable answers.

### Work through implications

When a stakeholder chooses behavior, ask follow-up implication questions immediately ("what if this arrives twice?", "what if timeout and success race?").

### Distinguish product from implementation

Translate implementation phrasing into behavior terms.

- "POST /x returns 404" -> "request is rejected as not found"
- "cron runs hourly" -> "timeout check becomes enabled each tick/window"

### Surface ambiguity explicitly

Do not silently choose fairness, ordering, or retry semantics. Record unresolved choices as open model questions.

### Use concrete examples

Drive examples with named actors/nodes/requests and specific sequences. Then map those sequences to actions.

### Iterate willingly

It is normal to revise state/action design after the first checker run.

### Know when to stop

Stop once model answers the scoped risk questions. Defer unrelated behavior to follow-up models.

## Common elicitation traps

### The "Obviously" trap

"Obviously" usually hides assumptions. Ask what failure would falsify the claim.

### The "Edge Case Spiral" trap

Do not enumerate infinite corner cases before establishing baseline model flow. Capture and queue them.

### The "Technical Solution" trap

If discussion collapses into tooling details, redirect to behavior and invariants.

### The "Vague Agreement" trap

Convert broad agreement into explicit guard/effect statements.

### The "Missing Actor" trap

Every transition needs a clear initiator: client, node, timer, or fault action.

### The "Equivalent Terms" trap

Resolve terminology collisions early (`job`, `task`, `command`) so actions/properties stay coherent.

## Elicitation session structure

Recommended session:

1. scope and risks (10-15 min)
2. state and actions (20-30 min)
3. properties and assumptions (15-20 min)
4. first scaffold + bounds review (10 min)

Deliverable template:

```text
Scope:
Included behavior:
Excluded behavior:
State fields:
Action catalog:
Safety properties:
Reachability properties:
Eventuality properties (if any):
Bounds:
Assumptions:
Open questions:
```

## After elicitation

Before handing to implementation:

1. ensure each critical risk has a corresponding safety property
2. run a first bounded checker pass
3. classify any failures immediately
4. capture counterexample traces as future regression inputs

## References

- [language reference](../../references/language-reference.md)
- [pattern catalog](../../references/patterns.md)
- [reusable-model signals](./references/library-spec-signals.md)
