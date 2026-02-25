# Quint authoring reference

This repository uses Quint (`.qnt`) as the specification language.

Quint models behaviour as state transitions:

- `var` declarations define mutable state.
- `action` declarations define valid next-state transitions.
- `val` and `temporal` declarations define safety/liveness checks.
- `run`, `test`, and `verify` execute and check the model.

Use this file as a practical reference for building maintainable models in this skill.

## File and module layout

A `.qnt` file is a module:

```quint
module Name {
  // declarations
}
```

Recommended declaration order:

1. `import`
2. `type`
3. `const`
4. `assume`
5. `var`
6. pure helpers (`pure def`, `val`)
7. actions (`init`, domain actions, `step`)
8. properties (`val` invariants, `temporal` formulas)
9. `run` scenarios

## Core declarations

### Constants and assumptions

```quint
const MAX_ATTEMPTS: int
assume PositiveAttempts = MAX_ATTEMPTS > 0
```

Use `const` for configuration-like values supplied externally.
Use `assume` to constrain constant domains.

### Mutable state

```quint
var attempts: str -> int
var locked: Set[str]
```

`var` declarations define the state that changes across steps.

### Types

```quint
type UserId = str
type Status = | Pending | Active | Locked
type Account = { status: Status, failedAttempts: int }
```

Use sum types for closed state machines and records for structured state.

### Pure helpers

```quint
pure def canLock(failedAttempts: int): bool =
  failedAttempts >= MAX_ATTEMPTS

val users = Set("alice", "bob")
```

Use `pure def` and `val` for reusable expressions that do not update state.

## Actions

Actions describe transitions. Primed variables (`x'`) represent next-state values.

### Single-variable update

```quint
action increment = n' = n + 1
```

### Guarded update (`all`)

```quint
action withdraw(user: str, amount: int) = all {
  amount > 0,
  balances.get(user) >= amount,
  balances' = balances.setBy(user, b => b - amount),
}
```

`all` means conjunction: every clause must hold.

### Alternative transitions (`any`)

```quint
action step = any {
  approve,
  reject,
  timeout,
}
```

`any` means non-deterministic choice among enabled actions.

### Non-deterministic bindings

```quint
action step = {
  nondet user = oneOf(USERS)
  nondet amount = oneOf(1.to(MAX_ATTEMPTS))
  any {
    loginFailure(user),
    loginSuccess(user),
  }
}
```

Use `nondet` bindings to pick values from finite domains.

### Stuttering

When an action should leave some state unchanged, assign it explicitly:

```quint
action noop = all {
  attempts' = attempts,
  locked' = locked,
}
```

## Data modeling patterns

### Sets

```quint
val active = Set("u1", "u2")
val hasU1 = active.contains("u1")
val withU3 = active.union(Set("u3"))
```

### Maps

```quint
val zeroes = USERS.mapBy(_ => 0)
val next = balances.setBy("alice", n => n + 1)
val aliceBalance = balances.get("alice")
```

### Records

```quint
val inv = { status: Pending, expiresAt: 42 }
val expired = { status: Expired, expiresAt: inv.expiresAt }
```

### Sequences (lists)

```quint
val q = List("a", "b")
val q2 = q.append("c")
```

## Canonical module skeleton

```quint
module Workflow {
  const USERS: Set[str]
  const MAX_RETRIES: int

  assume NonEmptyUsers = USERS.size() > 0
  assume PositiveRetries = MAX_RETRIES > 0

  var retries: str -> int
  var locked: Set[str]

  action init = all {
    retries' = USERS.mapBy(_ => 0),
    locked' = Set(),
  }

  action fail(user: str) = all {
    not(locked.contains(user)),
    retries' = retries.setBy(user, n => n + 1),
    locked' = if (retries.get(user) + 1 >= MAX_RETRIES)
      locked.union(Set(user))
      else locked,
  }

  action reset(user: str) = all {
    retries' = retries.set(user, 0),
    locked' = locked,
  }

  action step = {
    nondet user = oneOf(USERS)
    any {
      fail(user),
      reset(user),
    }
  }

  val retriesNonNegative = USERS.forall(u => retries.get(u) >= 0)
  val lockedBounded = locked.subseteq(USERS)

  temporal eventuallyQuiet = eventually(USERS.forall(u => retries.get(u) == 0))

  run smoke =
    init
      .then(5.reps(_ => step))
      .then(assert(retriesNonNegative))
      .then(assert(lockedBounded))
}
```

## Properties

### Safety invariants (`val`)

Use `val` boolean definitions for state predicates that must always hold.

```quint
val noNegativeBalance = ACCOUNTS.forall(a => balances.get(a) >= 0)
```

Typical safety checks:

- counters are non-negative
- status values are reachable/valid
- ownership relations stay consistent
- forbidden states never occur

### Temporal properties (`temporal`)

Use `temporal` formulas for eventual/progress requirements.

```quint
temporal eventuallySettled = eventually(ORDERS.forall(o => settled.get(o)))
```

Typical liveness checks:

- pending work eventually resolves
- retries do not loop forever
- queues are eventually drained

## Multi-module composition

Split domain areas into separate files and import what you need.

```quint
module BillingTests {
  import Billing.* from "billing"

  run billing_smoke =
    init.then(step).then(assert(nonNegativeTotals))
}
```

Use module boundaries to isolate concerns (auth, billing, notifications, policy).

## CLI workflow

```bash
# Structural checks
quint parse model.qnt
quint typecheck model.qnt

# Simulation and invariant checks
quint run model.qnt --invariants noNegativeBalance ownershipValid --max-steps=30

# Scenario/unit-style runs
quint test model.qnt --match smoke

# Model checking
quint verify model.qnt --invariant noNegativeBalance --max-steps=20
quint verify model.qnt --backend=tlc --invariant noNegativeBalance
```

## Style rules for this repository

- Keep action names in `lowerCamelCase`; module names in `PascalCase`.
- Model domain state explicitly; avoid implementation-specific fields.
- Prefer finite domains for model checking (`Set`, bounded `int` ranges).
- Keep `step` short; move logic into smaller actions.
- Name every important invariant (`val`) and check it in `run`/`verify`.
- Use comments only where intent is not obvious from names.

## Common mistakes

- Forgetting to constrain constants with `assume`.
- Leaving action branches unconstrained when state should remain unchanged.
- Encoding infra details (HTTP status codes, SQL internals) instead of domain behaviour.
- Defining large monolithic `step` actions that hide intent.
- Missing explicit liveness checks for long-running workflows.

## References

- Quint docs and CLI reference: https://quint-lang.org/

## Functional parity appendix

This appendix keeps the same example families as the original repository, expressed in Quint idioms.

### 1. Password auth: lockout and reset

```quint
type UserStatus = | Active | Locked | Deactivated
type TokenStatus = | NoToken | Pending | Used | Expired

action loginFailure(user: str, now: int) = all {
  status.get(user) == Active,
  failedAttempts' = failedAttempts.setBy(user, n => n + 1),
  status' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
    status.set(user, Locked)
    else status,
  lockedUntil' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
    lockedUntil.set(user, now + LOCKOUT_DURATION)
    else lockedUntil,
}

action requestPasswordReset(user: str, now: int) = all {
  Set(Active, Locked).contains(status.get(user)),
  resetTokenStatus' = resetTokenStatus.set(user, Pending),
  resetTokenExpiry' = resetTokenExpiry.set(user, now + RESET_TOKEN_EXPIRY),
}
```

### 2. RBAC: role checks and membership changes

```quint
type Role = | Viewer | Editor | Admin

pure def canWrite(user: str, workspace: str): bool =
  rolePermissions(membershipRole.get((workspace, user))).contains("documents.write")

action addMember(actor: str, workspace: str, newUser: str, role: Role) = all {
  canAdmin(actor, workspace),
  membershipRole' = membershipRole.set((workspace, newUser), role),
}

action createDocument(user: str, workspace: str) = all {
  canWrite(user, workspace),
  documentsPerWorkspace' = documentsPerWorkspace.setBy(workspace, n => n + 1),
}
```

### 3. Invitation lifecycle: pending, accept, decline, expire, revoke

```quint
type InviteStatus = | Pending | Accepted | Declined | Expired | Revoked

action invitationExpires(invId: int, now: int) = all {
  invites.get(invId).status == Pending,
  invites.get(invId).expiresAt <= now,
  invites' = invites.setBy(invId, i => {
    resource: i.resource, email: i.email, permission: i.permission,
    invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Expired
  }),
}
```

### 4. Soft delete and retention expiry

```quint
type DocStatus = | Active | Deleted | Purged

action deleteDocument(actor: str, doc: int, now: int) = all {
  status.get(doc) == Active,
  status' = status.set(doc, Deleted),
  deletedAt' = deletedAt.set(doc, now),
  deletedBy' = deletedBy.set(doc, actor),
}

action retentionExpires(doc: int, now: int) = all {
  status.get(doc) == Deleted,
  deletedAt.get(doc) + RETENTION_PERIOD <= now,
  status' = status.set(doc, Purged),
}
```

### 5. Notifications and digest batching

```quint
type NotificationKind =
  | Mention(commentId: int, mentionedBy: str)
  | Reply(replyId: int, originalCommentId: int, repliedBy: str)
  | Share(resource: str, sharedBy: str)
  | Assignment(taskId: int, assignedBy: str)
  | System(title: str)

action sendImmediateEmail(id: int) = all {
  used.contains(id),
  notification.get(id).emailStatus == Pending,
  preferenceFor(notification.get(id).user, notification.get(id).kind) == Immediately,
  notification' = notification.setBy(id, n => {
    user: n.user, kind: n.kind, status: n.status, emailStatus: Sent, createdAt: n.createdAt
  }),
}
```

### 6. Usage limits and quota reset

```quint
pure def maxApiRequests(p: Plan): int =
  match p { | Free => 100 | Pro => 10_000 | Enterprise => 1_000_000 }

action recordApiRequest(workspace: str) = all {
  apiRequestsToday.get(workspace) < maxApiRequests(plan.get(workspace)),
  apiRequestsToday' = apiRequestsToday.setBy(workspace, n => n + 1),
}

action resetDailyApiUsage(workspace: str, now: int) = all {
  nextResetAt.get(workspace) <= now,
  apiRequestsToday' = apiRequestsToday.set(workspace, 0),
  nextResetAt' = nextResetAt.set(workspace, now + 24),
}
```

### 7. Comments with mentions and reactions

```quint
action createReply(c: int, parent: int, u: str, mentionedUsers: Set[str]) = all {
  status.get(parent) == Active,
  author' = author.set(c, u),
  replyTo' = replyTo.set(c, parent),
  mentionsByComment' = mentionsByComment.set(c, mentionedUsers),
  replyEvents' = if (author.get(parent) != u and not(mentionedUsers.contains(author.get(parent))))
    replyEvents.union(Set((author.get(parent), c, parent)))
    else replyEvents,
}
```

### 8. Library integration boundaries (OAuth and billing)

```quint
module AppAuth {
  import OAuth.* from "oauth2"
  // local user policy reacts to auth/session lifecycle from imported module
}

module Billing {
  import StripeBilling.* from "stripe-billing"
  // local subscription policy reacts to payment/subscription lifecycle
}
```

## Legacy-to-Quint concept mapping (for migration work)

Use this mapping when converting legacy specs while keeping behavior.

| Earlier concept | Quint equivalent |
|-----------------|------------------|
| entity fields and relationships | `var` maps/sets indexed by identifiers |
| rule with `requires`/`ensures` | guarded `action` with primed updates |
| temporal trigger (`expires_at <= now`) | time-guarded action with `now` parameter/domain |
| trigger emission | explicit event set/queue updates |
| surface operation availability | operation guard predicates + action enabledness |
| config/default values | `const` + `assume`, or deterministic `init` values |
| black box function | `pure def` abstraction with constrained usage |
