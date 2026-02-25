---
name: p
description: Model and verify distributed event-driven systems in P. Velocity through executable clarity.
version: 1
auto_trigger:
  - file_patterns: ["**/*.p"]
  - file_patterns: ["**/*.pproj"]
  - keywords: ["p language", "p model", "p spec", "state machine", "model checking", "p check"]
---

# P

P is a state-machine language for modeling asynchronous distributed behavior and validating it with model checking.

## Routing table

| Task | Skill | When |
|------|-------|------|
| Writing or reading `.p` files | this skill | You need syntax, semantics, and modeling patterns |
| Building a model through conversation | `elicit` | Requirements are coming from stakeholders |
| Extracting a model from existing code | `distill` | Implementation exists and behavior must be recovered |

## Quick syntax summary

### Entity

P has no `entity` keyword. Equivalent domain entities are represented by typed tuples and machine state maps.

```p
type tOrder = (id: int, status: int, amount: int);
```

### External entity

External systems are modeled as boundary machines/events.

```p
event eGatewayReq: (reqId: int, amount: int, client: machine);
event eGatewayResp: (reqId: int, ok: bool);

machine GatewayBoundary {
  start state Ready {
    on eGatewayReq do (r: (reqId: int, amount: int, client: machine)) {
      send r.client, eGatewayResp, (reqId = r.reqId, ok = true);
    }
  }
}
```

### Value type

Use type aliases for value-like records.

```p
type tMoney = (currency: string, cents: int);
```

### Sum type

Use enum-tagged tuples for variant-like modeling.

```p
enum tNodeKind { BRANCH, LEAF }
type tNode = (kind: tNodeKind, data: int);
```

### Module given

Context is carried via machine `entry` payloads, local variables, and testcase drivers.

```p
machine Driver {
  var service: machine;

  start state Init {
    entry {
      service = new Service();
      new Client(service);
    }
  }
}
```

### Rule

Rule semantics map to event handlers inside machine states.

```p
on eReq do (r: tReq) {
  if (r.ok) { send r.client, eResp, (id = r.id, ok = true); }
}
```

### Trigger types

Common trigger classes in P models:

- external request event
- internal completion event
- timeout/scheduler event
- monitor-observed announcement

```p
event eReq: (reqId: int, client: machine);
event eDone: (reqId: int);
event eTimeout: (reqId: int);
event eObserved: (reqId: int);
```

### Rule-level iteration

Use loops and helper functions for collection-wide updates.

```p
fun ZeroAll(values: seq[int]): seq[int] {
  var out: seq[int];
  var i: int;
  out = values;
  i = 0;
  while (i < sizeof(out)) {
    out[i] = 0;
    i = i + 1;
  }
  return out;
}
```

### Ensures patterns

Postconditions are expressed by:

- state mutation
- emitted `send`/`announce` events
- monitor-observable outputs

```p
statusByReq[reqId] = 1;
send client, eResp, (reqId = reqId, ok = true);
announce eRequestCompleted, (reqId = reqId);
```

### Surface

P has no `surface` keyword; boundary contracts are modeled as event protocols + monitor properties.

```p
event eApiCreateReq: (reqId: int, actorId: int, value: int, client: machine);
event eApiCreateResp: (reqId: int, ok: bool, resourceId: int);
```

### Surface-to-implementation contract

Event payloads and monitor assertions form the contract between model intent and implementation behavior.

```p
spec EveryApiCreateResponds observes eApiCreateReq, eApiCreateResp {
  var pending: set[int];

  start state Idle {
    on eApiCreateReq goto Waiting with (r: (reqId: int, actorId: int, value: int, client: machine)) {
      pending += (r.reqId);
    }
  }

  hot state Waiting {
    on eApiCreateReq goto Waiting with (r: (reqId: int, actorId: int, value: int, client: machine)) {
      pending += (r.reqId);
    }
    on eApiCreateResp do (x: (reqId: int, ok: bool, resourceId: int)) {
      assert x.reqId in pending, "response without request";
      pending -= (x.reqId);
      if (sizeof(pending) == 0) goto Idle;
    }
  }
}
```

### Expressions

Use standard P expressions for arithmetic, comparisons, boolean logic, and collection operations (`sizeof`, `keys`, `values`, membership).

```p
assert req.amount > 0 && req.accountId in balanceByAccount, "invalid request";
if (sizeof(pending) == 0) { goto Idle; }
```

### Modular specs

Use modules to compose machines and attach monitor assertions.

```p
module Core = { Client, Server };
module Checked = assert SafetySpec, LivenessSpec in Core;
```

### Config

P has no dedicated config block; use `param`, driver initialization, and constants/helpers.

```p
param retryLimit: int;
```

### Defaults

Use `default(T)` for initialization/reset.

```p
var pending: set[int];
pending = default(set[int]);
```

### Deferred specs

For deferred behavior, keep placeholders and explicit TODO assumptions near machines/specs.

```p
// TODO: model compensation flow before enabling eCompensateReq path.
```

### Open questions

Record unresolved behavior explicitly in comments/issues adjacent to the model.

```p
// Open question: should retryLimit be per-tenant or global?
```

## Program outline

Recommended layout:

- `PSrc/`: events, datatypes, machines, shared functions
- `PSpec/`: monitor specs (`spec` machines)
- `PTst/`: test drivers and testcases
- `PForeign/`: optional foreign type/function implementations
- `<name>.pproj`: compiler project definition

## Checker workflow

```bash
p compile
p check
p check -tc tcSingleClient -s 100
```

## References

- [Language reference](./references/language-reference.md)
- [Patterns](./references/patterns.md)
- [Test generation](./references/test-generation.md)
