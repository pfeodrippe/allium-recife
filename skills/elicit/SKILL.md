---
name: elicit
description: Use when the user wants to build a P model from requirements, capture distributed behavior, or define monitors/tests through conversation.
---

# Elicitation

This guide builds executable P models from conversation with stakeholders.

## Scoping the specification

### Questions to ask first

1. What subsystem/workflow is in scope?
2. What is explicitly out of scope?
3. Which dependencies are external boundaries?
4. Is this greenfield or constrained by current implementation?

### Documenting scope decisions

Capture this first:

```text
Scope: subscription cancellation and refund workflow
Includes: cancellation request, refund decision, final state transition
Excludes: user signup, payment method capture, analytics exports
External boundaries: payment provider, notification provider
```

## Finding the right level of abstraction

### The "Why" test

Only include details that matter to behavioral correctness.

### The "Could it be different?" test

If implementation can change while behavior stays, abstract it.

### The "Template vs Instance" test

Model protocol categories first (`PaymentGateway`), then include vendor specifics only if behavior depends on them.

### Levels of abstraction

- too abstract: no executable transitions
- useful: explicit events, states, monitor obligations
- too concrete: framework routes/ORM details in core model

### Configuration vs hardcoding

Use test parameters and driver inputs when values may vary.

```p
param retryLimit: int;
```

### Black boxes

Model unknown internals as boundary helpers with explicit input/output obligations.

```p
fun RiskScore(reqId: int, amount: int): int;
```

## Elicitation methodology

### Phase 1: Scope definition

Outputs:

- boundary statement
- actor list
- core domain objects

### Phase 2: Happy path flow

Build first protocol skeleton from user story:

```p
event eCancelReq: (reqId: int, subId: int, client: machine);
event eCancelResp: (reqId: int, ok: bool);
```

### Phase 3: Edge cases and errors

Add explicit behavior for:

- timeout
- duplicate request
- out-of-order response
- unavailable dependency

### Phase 4: Refinement

Tighten machine boundaries and remove accidental implementation leakage.

### Phase 5: Property extraction and test scenarios

Convert requirements into monitors/tests:

```p
spec EveryCancelResponds observes eCancelReq, eCancelResp {
  var pending: set[int];
  start state Idle {
    on eCancelReq goto Waiting with (r: (reqId: int, subId: int, client: machine)) {
      pending += (r.reqId);
    }
  }
  hot state Waiting {
    on eCancelReq goto Waiting with (r: (reqId: int, subId: int, client: machine)) {
      pending += (r.reqId);
    }
    on eCancelResp do (x: (reqId: int, ok: bool)) {
      assert x.reqId in pending, "response without request";
      pending -= (x.reqId);
      if (sizeof(pending) == 0) goto Idle;
    }
  }
}
```

## Elicitation principles

### Ask one question at a time

Keep requirement extraction attributable.

### Work through implications

When one rule changes, ask what dependent rules/monitors/tests must change.

### Distinguish product from implementation

Translate route/framework talk into event/state obligations.

### Surface ambiguity explicitly

If unclear, mark open question instead of guessing.

### Use concrete examples

Prefer scenario examples over abstract prose.

### Iterate willingly

Models improve through repeated checker-guided refinement.

### Know when to stop

Stop when behavior is executable, checkable, and reviewed.

## Common elicitation traps

### The "Obviously" trap

Implicit assumptions about ordering/atomicity break under concurrency.

### The "Edge Case Spiral" trap

Do not lose core protocol clarity while over-expanding rare paths too early.

### The "Technical Solution" trap

Do not freeze framework choices before protocol behavior is modeled.

### The "Vague Agreement" trap

"Handle gracefully" must become explicit transitions/guards/properties.

### The "Missing Actor" trap

If no machine owns an obligation, model is incomplete.

### The "Equivalent Terms" trap

Normalize vocabulary early (`cancel` vs `revoke`, `request` vs `command`).

## Elicitation session structure

1. Scope + vocabulary
2. Event inventory
3. Machine/state skeleton
4. Error/timeout behavior
5. Monitor definitions
6. Testcase definitions
7. Open questions and next pass

## After elicitation

Checklist before handoff:

- each machine has one `start state`
- correlation IDs exist for request/response flows
- critical safety and liveness obligations are monitor-backed
- at least one testcase exercises each critical monitor

## References

- `../../references/language-reference.md`
- `../../references/patterns.md`
- `../../references/test-generation.md`
- `references/library-spec-signals.md`
