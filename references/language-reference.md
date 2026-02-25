# Language reference

This reference is P-first and intentionally broad so review workflows that previously depended on a large section map continue to work.

Canonical docs remain at https://p-org.github.io/P/. This file focuses on practical authoring guidance, examples, and local conventions.

## File structure

Recommended structure for projects in this repository:

```text
<project>/
  PSrc/
    Events.p
    Types.p
    Machines.p
    Modules.p
  PSpec/
    Safety.p
    Liveness.p
  PTst/
    Drivers.p
    Tests.p
  PForeign/                 # optional foreign implementations
  <Project>.pproj
```

### Formatting

- Keep declarations grouped by concern: events/types, machines, monitors, tests.
- Prefer short state handlers and helper functions over long, nested handlers.
- Keep monitor logic in `PSpec` unless tightly scoped local checks are unavoidable.
- Keep test drivers and test declarations together in `PTst`.

### Naming conventions

- Events: `eXxx` (e.g., `ePaymentAuthorized`).
- Types/enums: `tXxx` (e.g., `tRequest`, `tStatus`).
- Machines/specs/modules: `PascalCase`.
- Testcases: `tcXxx`.
- Variables/functions: `camelCase`.

## CLI setup

For compiling/checking models you need the upstream P CLI.

Prerequisites:

- .NET SDK 8.x
- Java runtime 11+

Install:

```bash
dotnet tool install --global P
```

Update:

```bash
dotnet tool update --global P
```

Verify:

```bash
p --help
```

Official references:

- https://p-org.github.io/P/getstarted/install/
- https://p-org.github.io/P/getstarted/usingP/

## Module given

P does not have a `given` keyword. Equivalent context is carried by:

- machine-local variables initialized in `entry`
- machine creation payloads
- module composition and replacement
- testcase `main` driver construction

```p
machine Driver {
  var service: machine;

  start state Init {
    entry {
      service = new PaymentService();
      new Client(service);
    }
  }
}
```

## Entities

P has no `entity` declaration. Domain entities are typically represented by:

- tuple aliases (`type`)
- machine state maps/sets/sequences
- event payloads carrying entity IDs and snapshots

### External entities

Model external systems as boundary machines and boundary events.

```p
event eGatewayCharge: (reqId: int, amount: int, client: machine);
event eGatewayResult: (reqId: int, ok: bool);

machine GatewayBoundary {
  start state Ready {
    on eGatewayCharge do (r: (reqId: int, amount: int, client: machine)) {
      // abstraction: always succeeds in this model profile
      send r.client, eGatewayResult, (reqId = r.reqId, ok = true);
    }
  }
}
```

### Internal entities

Use tuple aliases and indexed machine state.

```p
type tAccount = (id: int, balance: int, locked: bool);

machine Ledger {
  var accounts: map[int, tAccount];

  start state Ready {}
}
```

### Value types

Use `type` aliases for value-like records.

```p
type tMoney = (currency: string, cents: int);
type tTransfer = (id: int, from: int, to: int, amount: tMoney);
```

### Sum types

P does not provide algebraic sum types directly. Use enum-tagged tuples.

```p
enum tCommandKind { CREATE, UPDATE, DELETE }

type tCommand = (kind: tCommandKind, id: int, payload: int);
```

### Field types

Common field types include:

- primitives: `int`, `bool`, `string`, `float`, `machine`
- collections: `set[T]`, `seq[T]`, `map[K, V]`
- nested tuples and enums

### Relationships

Represent relationships through identifiers and maps.

```p
var ownerBySession: map[int, int];
var commentsByPost: map[int, seq[int]];
```

### Projections

Use helper functions to project derived views.

```p
fun IsActive(a: tAccount): bool {
  return !a.locked;
}
```

### Derived values

Derived values are computed when needed, usually in helpers.

```p
fun Remaining(limit: int, used: int): int {
  return limit - used;
}
```

## Rules

P has no `rule` keyword. Equivalent behavior is encoded in event handlers in machine states.

### Rule structure

Equivalent decomposition:

- trigger: `on eX ...`
- preconditions: `if`, `assert`
- postconditions: state mutation + emitted events

```p
event eWithdrawReq: (reqId: int, accountId: int, amount: int, client: machine);
event eWithdrawResp: (reqId: int, ok: bool, balance: int);

machine Bank {
  var balanceByAccount: map[int, int];

  start state Ready {
    on eWithdrawReq do (r: (reqId: int, accountId: int, amount: int, client: machine)) {
      var current: int;
      current = balanceByAccount[r.accountId];

      if (r.amount <= current) {
        balanceByAccount[r.accountId] = current - r.amount;
        send r.client, eWithdrawResp,
          (reqId = r.reqId, ok = true, balance = balanceByAccount[r.accountId]);
      } else {
        send r.client, eWithdrawResp,
          (reqId = r.reqId, ok = false, balance = current);
      }
    }
  }
}
```

### Rule-level iteration

Use loops and helper functions for collection-wide effects.

```p
fun ZeroOut(xs: seq[int]): seq[int] {
  var i: int;
  var out: seq[int];
  i = 0;
  out = xs;

  while (i < sizeof(out)) {
    out[i] = 0;
    i = i + 1;
  }

  return out;
}
```

### Multiple rules for the same trigger

In P, this corresponds to:

- handling the same event in multiple states, or
- delegating to different machines/modules.

```p
machine Worker {
  start state Idle {
    on eStart goto Busy;
  }

  state Busy {
    on eStart do { /* ignored or counted while busy */ }
  }
}
```

### Trigger types

Common trigger categories:

- external stimuli (`on eApiReq ...`)
- internal completion (`on eDone ...`)
- timeout/scheduler (`on eTimeout ...`)
- observed boundary events (`announce eX` + monitor `observes eX`)

### Preconditions (requires)

Encode preconditions using `if` and `assert`.

```p
assert reqId in pending, "response without request";
if (!(userId in activeUsers)) {
  send client, eResp, (reqId = reqId, ok = false);
  return;
}
```

### Local bindings (let)

Use local variables inside handlers/functions.

```p
var next: int;
next = current + 1;
```

### Discard bindings

If payload fields are not needed, ignore by convention and use only required fields.

```p
on eHeartbeat do (hb: (nodeId: int, ts: int, seq: int)) {
  // only hb.nodeId is relevant in this state
}
```

### Postconditions (ensures)

Represent by:

- state updates
- outgoing events/announcements
- monitor-visible side effects

```p
statusById[id] = COMPLETED;
announce eJobCompleted, (id = id);
```

## Expression language

### Navigation

Tuple field access:

```p
resp.reqId
req.amount
```

### Join lookups

Map lookup + relation traversal by IDs.

```p
if (sessionId in userBySession) {
  userId = userBySession[sessionId];
}
```

### Collection operations

`sizeof`, `keys`, `values`, membership (`in`), set/map add/remove semantics.

```p
pending += (reqId);
pending -= (reqId);
assert sizeof(pending) >= 0;
```

### Comparisons

`==`, `!=`, `<`, `<=`, `>`, `>=`.

### Arithmetic

`+`, `-`, `*`, `/`, modulo where supported.

### Boolean logic

`&&`, `||`, `!`.

### Conditional expressions

Use statements:

```p
if (ok) {
  goto Success;
} else {
  goto Failure;
}
```

### Existence

```p
if (id in stateById) { ... }
if (sizeof(q) == 0) { ... }
```

### Literals

```p
42
true
"hello"
(reqId = 1, ok = true)
```

### Black box functions

Declare foreign/abstract functions when implementation detail is intentionally hidden.

```p
fun ChooseShard(accountId: int): int;
```

### The `with` and `where` keywords

P supports `with` in goto handlers:

```p
on eReq goto Waiting with (r: (reqId: int)) {
  pending += (r.reqId);
}
```

P does not use `where` for collection projections in core language syntax.

### Entity collections

Equivalent to `map`, `set`, `seq` collections encoding domain entities.

## Deferred specifications

For behavior intentionally postponed:

- keep placeholder events/machines/specs
- record explicit TODO assumptions
- avoid silent omission

```p
// TODO: model compensation workflow before enabling this path.
```

## Open questions

Track unresolved behavior near code and in issue tracker references.

```p
// Open question: should retries remain bounded at 3 or be policy-controlled?
```

## Config

P has no dedicated config block. Common alternatives:

- `param` declarations in tests
- driver payload initialization
- helper function constants

```p
param nClients: int;
param retryLimit: int;
```

## Defaults

Use `default(T)` for zero-value initialization/reset.

```p
var s: set[int];
var m: map[int, int];
s = default(set[int]);
m = default(map[int, int]);
```

## Modular specifications

### Namespaces

Separate by folders/files and module names.

### Using other specs

Compose monitors into checked modules.

```p
module Core = { Client, Service };
module Checked = assert SafetySpec, LivenessSpec in Core;
```

### Referencing external entities and triggers

Model external boundaries explicitly with events and boundary machines.

### Responding to external triggers

External requests are regular events consumed by boundary-facing machines.

### Configuration

Use parameterized tests and driver payloads for scenario variability.

### Breaking changes

When changing payload schemas or event names:

- migrate all handlers/monitors/tests together
- keep temporary compatibility adapters only when necessary

### Local specs

Use local monitors for feature-specific constraints when global monitors become too broad.

## Surfaces

P has no `surface` construct. Equivalent boundary contracts are represented by:

- event protocol shape
- acceptance/rejection semantics
- monitor properties over boundary events

### Actor declarations

Actors appear as machine refs or actor IDs in payloads.

```p
event eCreateReq: (reqId: int, actorId: int, value: int, client: machine);
```

### Surface structure

Equivalent structure in P terms:

- **exposes**: response event payload fields
- **provides**: accepted request events
- **guarantees**: monitor properties

### Examples

```p
event eApiCreateReq: (reqId: int, actorId: int, value: int, client: machine);
event eApiCreateResp: (reqId: int, ok: bool, resourceId: int);

machine ApiBoundary {
  var nextId: int;

  start state Ready {
    entry { nextId = 1; }

    on eApiCreateReq do (r: (reqId: int, actorId: int, value: int, client: machine)) {
      var createdId: int;
      if (r.value <= 0) {
        send r.client, eApiCreateResp, (reqId = r.reqId, ok = false, resourceId = 0);
        return;
      }

      createdId = nextId;
      nextId = nextId + 1;
      send r.client, eApiCreateResp, (reqId = r.reqId, ok = true, resourceId = createdId);
      announce eResourceCreated, (reqId = r.reqId, resourceId = createdId, actorId = r.actorId);
    }
  }
}

event eResourceCreated: (reqId: int, resourceId: int, actorId: int);

spec ApiOnlyCreatesPositiveValues observes eApiCreateReq, eApiCreateResp {
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

    on eApiCreateResp do (r: (reqId: int, ok: bool, resourceId: int)) {
      assert r.reqId in pending, "response without request";
      pending -= (r.reqId);
      if (sizeof(pending) == 0) {
        goto Idle;
      }
    }
  }
}
```

## Validation rules

This repository expects the following for review-ready models:

1. every machine has exactly one `start state`
2. all referenced events/types/functions are declared
3. critical safety and liveness obligations are encoded as monitors
4. tests include nominal and adversarial scenarios
5. correlation IDs exist where concurrent request-response matching matters
6. checker schedule budgets are intentional (`-s` strategy documented)

## Anti-patterns

- synchronous assumptions over asynchronous event queues
- request/response protocols without IDs
- giant multi-purpose machines spanning unrelated domains
- monitor-free critical behavior
- overuse of `ignore` for events that should be deferred/handled
- shallow checker runs (`-s 1`) treated as sufficient for complex flows

## Glossary

- **machine**: concurrent state machine with private event queue
- **spec**: observer machine asserting safety/liveness
- **hot state**: liveness-sensitive monitor state
- **module**: composition/replacement expression over machines
- **testcase**: checker scenario declaration
- **schedule**: one interleaving explored by `p check`
