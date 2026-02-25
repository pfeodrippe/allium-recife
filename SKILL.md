---
name: quint
description: Executable specifications for reliable systems with Quint. Velocity through clarity and model checking.
version: 1
auto_trigger:
  - file_patterns: ["**/*.qnt"]
  - keywords: ["quint", "quint spec", "quint specification", ".qnt file", "model checker"]
---

# Quint

Quint is an executable specification language for describing state machines and checking behavioural properties with simulation and model checking.

Key principles:

- Model behaviour, not implementation details
- Keep state explicit and transitions precise
- Check safety with invariants early
- Add temporal properties when needed
- Use counterexamples to refine design

Quint does NOT prescribe framework choices, database schemas, API shapes, or UI layout unless these are domain constraints you intentionally model.

## Routing table

| Task | Skill | When |
|------|-------|------|
| Writing or reading `.qnt` files | this skill | You need Quint syntax and modeling structure |
| Building a spec through conversation | `elicit` | User is defining behaviour and constraints |
| Extracting a spec from existing code | `distill` | User has implementation code and wants a Quint model |

## Quick syntax summary

### Module and declarations

```quint
module Bank {
  const ADDRESSES: Set[str]
  var balances: str -> int

  assume ValidAddresses = ADDRESSES.size() > 0
}
```

### Actions (`init`, `step`)

```quint
module Counter {
  var n: int

  action init = n' = 0

  action step = all {
    n < 10,
    n' = n + 1,
  }
}
```

### Action choice (`any`) and guarded transitions (`all`)

```quint
module Workflow {
  var status: str

  action init = status' = "pending"

  action approve = all {
    status == "pending",
    status' = "approved",
  }

  action reject = all {
    status == "pending",
    status' = "rejected",
  }

  action step = any {
    approve,
    reject,
  }
}
```

### Non-deterministic choice

```quint
module Scheduler {
  var slot: int

  action init = slot' = 0

  action step = {
    nondet nextSlot = oneOf(1.to(5))
    slot' = nextSlot
  }
}
```

### Invariants and temporal properties

```quint
module BankProps {
  const ADDRESSES: Set[str]
  var balances: str -> int

  action init = balances' = ADDRESSES.mapBy(_ => 0)
  action step = balances' = balances

  val no_negatives =
    ADDRESSES.forall(a => balances.get(a) >= 0)

  temporal eventual_zero =
    eventually(ADDRESSES.forall(a => balances.get(a) == 0))
}
```

### Runs for executable scenarios

```quint
module CounterRun {
  var x: int

  action init = x' = 0
  action step = x' = x + 1

  run grows_to_three =
    init.then(3.reps(_ => step)).then(assert(x == 3))
}
```

### Imports

```quint
module TestCounter {
  import Counter.* from "counter"

  run smoke = init.then(2.reps(_ => step)).then(assert(n == 2))
}
```

## Common command loop

```bash
# Install CLI
npm i -g @informalsystems/quint

# Parse and typecheck
quint parse model.qnt
quint typecheck model.qnt

# Quick simulation checks (invariants)
quint run model.qnt --invariants no_negatives progress_ok --max-steps=30

# Model checking (Apalache by default; TLC also supported)
quint verify model.qnt --invariant no_negatives --max-steps=20
quint verify model.qnt --backend=tlc --invariant no_negatives

# Run scenario/unit tests
quint test model.qnt --match test
```

## Modeling guidance

- Keep `init` and `step` explicit in the main module.
- Prefer small composable actions, then combine in `step` with `any`.
- Give each key domain guarantee a named `val` invariant.
- Use `run` definitions for concrete examples and regression scenarios.
- Separate reusable modules by domain (auth, billing, notifications) and import them.

## Functional parity targets

When migrating from the original material, preserve behavior for:

- Password auth with reset token lifecycle
- RBAC membership/permission changes
- Resource invitations and share revocation
- Soft-delete and retention expiry
- Notification preferences with digest batching
- Usage limits and daily quota reset
- Comments, mentions, and reactions
- OAuth and billing integration boundaries

## Reference

- Quint language manual: `references/language-reference.md`
- Reusable patterns: `references/patterns.md`
- Test and MBT guidance: `references/test-generation.md`
