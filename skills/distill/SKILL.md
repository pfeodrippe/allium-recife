---
name: distill
description: Use when the user has existing implementation code and wants to reverse-engineer a P model with machines, monitors, modules, and tests.
---

# Distillation guide

This guide extracts executable P models from existing implementation code.

Primary goal: produce an accurate behavioral model with explicit safety/liveness obligations.

## Scoping the distillation effort

### Questions to ask first

1. Which subsystem is in scope?
2. Which code paths are out-of-scope (legacy, experimental, sunset)?
3. Is target behavior "as implemented" or "as intended"?

### The "Would we rebuild this?" test

If a path exists only due to migration debt and would not be intentionally rebuilt, do not elevate it as core model behavior without explicit agreement.

### Documenting scope decisions

Record this before extraction:

```text
Scope: payout orchestration
Includes: request intake, provider attempt, retry, terminal response
Excludes: admin dashboard, analytics sinks, invoice PDF rendering
External boundaries: payout provider, notification provider
```

## Finding the right level of abstraction

### The "Why" test

For each observed implementation detail ask: why does system correctness depend on this?

### The "Could it be different?" test

If yes, likely implementation detail. If no, likely behavioral requirement.

### The "Template vs Instance" test

Separate protocol shape from vendor implementation details.

- Template: `PaymentGateway`, `AuthProvider`
- Instance: specific vendor SDK calls

## The distillation mindset

### Code is over-specified

Implementation usually includes:

- framework routing boilerplate
- ORM/database call structure
- transport/retry middleware mechanics

Model should preserve behavior, not framework artifact shape.

### Ask "Would a product owner care?"

If not, do not force it into core model vocabulary.

### Distinguish means from ends

- Means: `POST /x`, queue client API, ORM transaction wrapper
- Ends: accepted/rejected transitions, eventual response obligations, invariant preservation

## The concrete detail problem

### Google OAuth example

Keep provider details only when user-visible behavior or policy depends on provider-specific semantics.

### Database choice example

Storage engine details are typically not model-level semantics unless consistency guarantees are engine-dependent and behaviorally relevant.

### Third-party integration example

When integration protocol repeats across domains, extract reusable module boundary.

### The "Multiple implementations" heuristic

If two different implementations satisfy the same protocol obligations, model those obligations and avoid encoding implementation internals.

## Distillation process

### Step 1: Map the territory

Collect:

- inbound request handlers
- outbound events/effects
- async jobs/schedulers
- retries/timeouts

### Step 2: Extract entity states

Find implicit lifecycle states hidden in flags/conditionals.

Example hidden lifecycle:

```text
pending -> processing -> succeeded/failed -> compensated
```

### Step 3: Extract transitions

Map implementation branches to explicit event-driven transitions.

```p
on ePayoutRequested do (...) { ... goto WaitingProvider; }
on eProviderResult do (...) { if (ok) goto Completed; else goto Retrying; }
```

### Step 4: Find temporal triggers

Identify timeout/retry/cron-driven behavior and model as explicit events.

```p
event eRetryTimeout: (reqId: int);
event eDailyReconcileTick;
```

### Step 5: Identify external boundaries

Model dependencies as boundary machines/events with explicit assumptions.

### Step 6: Abstract away implementation

Drop route names, table names, concrete SDK calls unless behavior depends on them.

### Step 7: Validate with stakeholders

Review extracted model with owners to classify divergence as:

- `model bug`
- `code bug`
- `intentional gap`
- `aspirational`

## Recognising library spec candidates

### Signals in the code

- same integration protocol in multiple services
- repeated timeout/retry policy for same external system
- same idempotency/dedup patterns duplicated

### Questions to ask

1. Could this protocol be reused in another domain/service?
2. Are customization points explicit and small?
3. Can app-domain policy be separated from integration mechanics?

### How to handle

- reuse existing reusable module if available
- extract new module if protocol is generic and repeated
- keep inline if deeply domain-specific

### Red flags: integration logic in your spec

- one machine mixes domain state with multiple external provider lifecycle states
- monitors are copied with only event name substitutions

### Common library spec extractions

- auth protocol
- payment protocol
- webhook verification + retry
- queue retry/dead-letter protocol

## Common distillation challenges

### Challenge: Duplicate terminology

Normalize naming before modeling events/states.

### Challenge: Implicit state machines

Lift implicit states into explicit machine states.

### Challenge: Scattered logic

Unify behavior split across handlers, workers, and schedulers.

### Challenge: Dead code and historical accidents

Avoid canonizing accidental behavior.

### Challenge: Missing error handling

If implementation lacks explicit timeout/compensation behavior, mark it as open risk and model expected guardrails.

### Challenge: Over-engineered abstractions

Prefer behavior-centric boundaries to framework-centric layers.

## Checklist: Have you abstracted enough?

- protocol events are explicit
- IDs/correlation are explicit
- monitor obligations are explicit
- framework artifacts are minimized

## Checklist: Terminology consistency

- one concept, one term
- consistent event naming
- state names carry domain meaning

## After distillation

Deliverables:

- `PSrc/*.p` (events/types/machines/modules)
- `PSpec/*.p` (safety/liveness monitors)
- `PTst/*.p` (drivers/tests)
- `<Project>.pproj`
- divergence report with classification

Suggested handoff summary:

```text
Distilled: payout workflow
Machines: PayoutCoordinator, ProviderBoundary, RetryScheduler
Monitors: NoDoublePayout, EveryRequestResponds
Tests: tcNominal, tcProviderFailure, tcRetryExhausted
Open issues: provider duplicate callback policy unresolved
```

## References

- `../../references/language-reference.md`
- `../../references/patterns.md`
- `../../references/test-generation.md`
- `references/worked-examples.md`
