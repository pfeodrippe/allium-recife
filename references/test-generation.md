# Test generation

## Checker planning

From a P model, generate testcases that exercise both protocol behavior and monitor obligations.

## 1. Contract tests per protocol

For each request/response protocol, generate:

- success path
- rejection path
- malformed/out-of-order path

Template:

```p
test tcContractNominal [main=ContractDriver]:
  CheckedSystem;

test tcContractRejected [main=RejectedDriver]:
  CheckedSystem;
```

## 2. State transition tests

For each lifecycle machine:

- valid transitions
- invalid transition attempts
- terminal-state behavior

Example target states:

```text
PENDING -> ACTIVE -> COMPLETED
PENDING -> CANCELLED
COMPLETED (terminal)
```

## 3. Temporal and retry tests

For timeout/retry behavior, include:

- before-timeout path
- at-timeout path
- retry-exhaustion path

Template events:

```p
event eTimeout: (reqId: int);
event eRetryExhausted: (reqId: int);
```

## 4. Monitor-driven tests

For each `spec` monitor:

- at least one testcase expected to satisfy the property
- at least one adversarial testcase that would violate property if guards regress

Monitor template:

```p
spec EveryRequestResponds observes eReq, eResp {
  var pending: set[int];
  start state Idle {
    on eReq goto Waiting with (r: (reqId: int, client: machine)) {
      pending += (r.reqId);
    }
  }
  hot state Waiting {
    on eReq goto Waiting with (r: (reqId: int, client: machine)) {
      pending += (r.reqId);
    }
    on eResp do (x: (reqId: int, ok: bool)) {
      assert x.reqId in pending, "response without request";
      pending -= (x.reqId);
      if (sizeof(pending) == 0) goto Idle;
    }
  }
}
```

## 5. Parameterized coverage

When behavior depends on scale/config, generate parameterized tests.

```p
param nClients: int;
param retryLimit: int;

test param (nClients in [1, 2, 4], retryLimit in [1, 2, 3])
  assume (nClients * retryLimit <= 8)
  tcScale [main=ScaleDriver]:
  CheckedSystem;
```

## 6. Concurrency and race tests

Create drivers that force risky interleavings:

- duplicate command events
- response-before-request bugs
- timeout-vs-success races
- revoke-vs-authorize races

## 7. Drift-focused regression tests

When drift is found:

- add testcase reproducing current code behavior
- add testcase reproducing current model behavior
- decide source of truth, then keep the aligned one

## 8. Checker profile matrix

Use explicit schedule budgets:

1. smoke: `p check -tc <x> -s 1`
2. local: `p check -tc <x> -s 100`
3. deep: `p check -tc <x> -s 10000`

## 9. Reporting checklist

Record with each run:

- testcase name
- schedule count
- seed/strategy metadata
- pass/fail outcome
- failing trace summary

## 10. Minimal acceptance bar

Treat a change as test-ready when:

- touched monitors have at least one exercising testcase
- at least one adversarial testcase exists for changed behavior
- at least one run at `-s >= 100` passes
- no unresolved assertion/deadlock/unhandled-event findings remain
