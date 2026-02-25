---
name: elicit
description: This skill should be used when the user wants to build a Quint spec, elicit requirements, capture domain behaviour, or specify a feature as a `.qnt` model.
---

# Elicitation

This skill runs structured conversations that produce executable Quint models.

Goal: move from stakeholder language to explicit state transitions with checkable invariants.

## What success looks like

A successful elicitation session produces a `.qnt` model that has:

1. Bounded domains (`const` + `assume`) so checks are tractable.
2. Explicit lifecycle state (`type` + `var` maps/sets).
3. Guarded transitions (`action ... = all { ... }`).
4. A composed transition relation (`action step = any { ... }`).
5. Named safety properties (`val`) and at least one temporal property when applicable.
6. `run` scenarios for happy path and failure path.

## Session protocol

### Phase 0: frame the session

Ask:

1. What decision should this model help us make now?
2. What is explicitly out of scope for this pass?
3. Which guarantees are non-negotiable (security, money, compliance, SLA)?

Record scope at top of file:

```quint
module FeatureModel {
  // Scope: invitation lifecycle and access propagation
  // Includes: invite creation, accept/decline/revoke/expiry
  // Excludes: authentication, billing, analytics
}
```

### Phase 1: identify nouns and lifecycles

Extract domain nouns as future state keys.

Ask:

1. What are the primary entities in this workflow?
2. What statuses can each entity hold?
3. Which statuses are terminal?
4. Which transitions must never occur?

Convert to Quint structures:

```quint
type InviteStatus = | Pending | Accepted | Declined | Expired | Revoked
var inviteStatus: int -> InviteStatus
```

### Phase 2: identify triggers and guards

For each desired transition ask:

1. What triggers it?
2. Which preconditions must hold?
3. Which state changes atomically?
4. Which side effects/events are observable?

Template:

```quint
action acceptInvitation(invId: int, user: str) = all {
  inviteStatus.get(invId) == Pending,
  inviteEmail.get(invId) == user,
  inviteStatus' = inviteStatus.set(invId, Accepted),
  shareActive' = shareActive.set((inviteResource.get(invId), user), true),
}
```

### Phase 3: time, limits, and policy values

Force explicit timing and thresholds.

Ask:

1. What expires and when?
2. What retries and with what cap?
3. What resets and on what schedule?
4. What is configurable vs fixed policy?

Template:

```quint
const INVITATION_EXPIRY: int
const NOW_VALUES: Set[int]

action invitationExpires(invId: int, now: int) = all {
  inviteStatus.get(invId) == Pending,
  inviteExpiresAt.get(invId) <= now,
  inviteStatus' = inviteStatus.set(invId, Expired),
}
```

### Phase 4: failure paths and blocked operations

Most hidden bugs are in disabled paths.

Ask:

1. What should happen when a guard fails?
2. Is state unchanged, or does an explicit failure event occur?
3. Which blocked actions must be user-visible?

Template:

```quint
action downgradeBlocked(workspace: str, newPlan: Plan) = all {
  documents.get(workspace) > maxDocuments(newPlan)
    or members.get(workspace) > maxMembers(newPlan),
  plan' = plan,
  documents' = documents,
  members' = members,
}
```

### Phase 5: concurrency and ordering

Ask:

1. If two actors race, what outcomes are valid?
2. Are operations commutative or order-sensitive?
3. Should one action disable another immediately?

Capture with additional guards and invariants.

### Phase 6: property extraction

Turn requirements into checks.

- Safety: "never negative", "never unauthorized", "never two active owners".
- Liveness: "pending eventually resolves", "queue eventually drains".

Template:

```quint
val attemptsNonNegative = USERS.forall(u => failedAttempts.get(u) >= 0)

val activeShareImpliesAcceptedInvite =
  SHARES.forall(s => if (shareActive.get(s)) inviteStatus.get(shareInvite.get(s)) == Accepted else true)

temporal eventuallyNoPending = eventually(INVITES.forall(i => inviteStatus.get(i) != Pending))
```

### Phase 7: scenario runs

Define at least one run per major flow.

```quint
run invite_happy_path =
  init
    .then(createInvite(1, "doc-1", "alice", 0))
    .then(acceptInvitation(1, "alice"))
    .then(assert(inviteStatus.get(1) == Accepted))
```

## Abstraction filters (use continuously)

### Why test

If stakeholders do not care about a detail, likely implementation detail.

- Keep: lockout duration, trial reminder lead time, role permissions.
- Drop: ORM entities, HTTP route strings, queue names.

### Could-it-be-different test

If behavior could change without changing product intent, abstract it.

### Policy test

If legal/security/billing policy depends on it, model it explicitly.

## Pattern-specific elicitation question banks

Use these when the team has fuzzy requirements.

### Password auth with reset

1. Exactly when is a user considered locked?
2. Does successful login clear counters?
3. Can locked users request reset?
4. Does reset unlock account?
5. How do tokens expire and invalidate prior tokens?

### RBAC

1. Which roles exist and what do they inherit?
2. Who can assign/revoke each role?
3. Can owners remove themselves?
4. Are role changes immediate for active sessions?
5. Which operations require read/write/admin separately?

### Resource invitations

1. Which permissions can be invited?
2. Who is allowed to invite/revoke?
3. Can non-members accept invites?
4. What happens on expiry?
5. Can an email have multiple pending invites to same resource?

### Soft delete and restore

1. What is soft-delete retention duration?
2. Who can restore?
3. Is restore allowed after retention expiry?
4. What is hard-delete trigger?
5. How should "empty trash" behave?

### Notifications and digests

1. Which notification kinds exist?
2. Which are immediate vs digest vs never per user preference?
3. How is digest window scheduled?
4. What avoids duplicate mention/reply notifications?
5. What is archive semantics?

### Usage limits and quotas

1. Which limits are plan-dependent?
2. Which actions are blocked at limit?
3. Which counters reset and when?
4. How are downgrades blocked when over limit?
5. What overage behavior is allowed?

### Comments and mentions

1. Max reply depth?
2. Can deleted comments receive replies/reactions?
3. How are mentions parsed and updated on edit?
4. When should reply author notification be suppressed?
5. Which moderation operations are admin-only?

### OAuth and billing integration boundaries

1. Which behaviors belong in imported module vs local module?
2. What external events are required for local transitions?
3. How are suspended users handled on auth success?
4. Which billing events mutate subscription state?
5. What local invariants must still hold despite external events?

## Canonical output template

Use this scaffold for first deliverable.

```quint
module Feature {
  import External.* from "external"

  const IDS: Set[int]
  const USERS: Set[str]
  const NOW_VALUES: Set[int]
  const POLICY_LIMIT: int

  assume NonEmptyUsers = USERS.size() > 0
  assume PositiveLimit = POLICY_LIMIT > 0

  type Status = | Pending | Active | Closed

  var status: int -> Status
  var owner: int -> str
  var counter: int -> int

  action init = all {
    status' = IDS.mapBy(_ => Pending),
    owner' = IDS.mapBy(_ => ""),
    counter' = IDS.mapBy(_ => 0),
  }

  action activate(id: int, user: str) = all {
    status.get(id) == Pending,
    status' = status.set(id, Active),
    owner' = owner.set(id, user),
    counter' = counter,
  }

  action close(id: int) = all {
    status.get(id) == Active,
    status' = status.set(id, Closed),
    owner' = owner,
    counter' = counter,
  }

  action step = {
    nondet id = oneOf(IDS)
    nondet user = oneOf(USERS)
    any {
      activate(id, user),
      close(id),
    }
  }

  val countersNonNegative = IDS.forall(id => counter.get(id) >= 0)

  run smoke = init.then(step).then(assert(countersNonNegative))
}
```

## Quality bar checklist

Before ending session confirm:

1. Every status has at least one incoming transition.
2. Every non-terminal status has at least one outgoing transition.
3. Every time-based rule has explicit guard and threshold source.
4. Every critical user operation has a blocked-path behavior.
5. Every major entity lifecycle has at least one invariant.
6. At least one run covers happy path and one covers rejection/failure.

## Common failure modes

- UI flow modeled as state instead of domain state.
- Missing blocked/forbidden transitions.
- No explicit policy constants.
- No integration boundary (external behavior mixed into core module).
- Invariants too weak to catch drift.

## Hand-off guidance

- Use `tend` for targeted edits after initial model exists.
- Use `distill` when behavior must be extracted from code.
- Use `weed` for model-vs-code drift analysis.

## References

- `../../references/language-reference.md`
- `../../references/patterns.md`
- `../../references/test-generation.md`
- `references/library-spec-signals.md`
