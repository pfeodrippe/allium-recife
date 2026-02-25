---
name: distill
description: This skill should be used when the user has existing code and wants to extract a Quint specification (`.qnt`) from implementation behaviour.
---

# Distillation guide

Distillation converts implementation behavior into executable Quint models.

Goal: preserve behavior guarantees while discarding implementation mechanics.

## Deliverables

A complete distillation pass outputs:

1. One or more `.qnt` modules with explicit state and actions.
2. Named invariants for critical guarantees.
3. `run` scenarios that reproduce key production paths.
4. Notes on assumptions/ambiguities and unresolved decisions.

## Step-by-step workflow

### Step 1: scope and boundaries

Decide what is in this pass.

Ask:

1. Which services/modules are in scope?
2. Which routes/jobs/consumers represent domain behavior?
3. Which integrations should be imported modules?
4. Which legacy paths are excluded?

Write scope comments in model header.

### Step 2: inventory lifecycle state

Extract statuses and state fields from code, schema, and tests.

Capture:

- status enums/constants
- counter fields
- timestamps/deadlines
- ownership/permission relations
- event/outbox records

Map these to `type`, `var`, and indexed maps.

### Step 3: inventory transition sources

List all behavior entry points:

- API handlers
- background jobs/cron
- queue consumers
- webhook handlers
- internal service methods with business guards

Each candidate becomes one or more Quint `action`s.

### Step 4: extract guards and effects

For each transition source, record:

1. Entry trigger.
2. Guard predicates.
3. State updates.
4. Side effects (events/messages/notifications).
5. Error path semantics.

Then encode as guarded actions.

### Step 5: extract temporal behavior

Look for:

- expirations
- retries/backoff
- lockout windows
- daily resets
- trial reminders

Represent time with bounded constants + explicit `now` domains.

### Step 6: encode integration boundaries

When behavior belongs to provider mechanics, split into imported module references.

Example:

```quint
module AppAuth {
  import OAuth.* from "oauth2"
  // local account policy reacts to external auth/session lifecycle
}
```

### Step 7: add properties and runs

Add invariants from implicit assumptions in code.

- counters non-negative
- role and ownership consistency
- status progression constraints
- no invalid terminal transitions

Add `run` scenarios for:

- happy path
- one blocked path
- one timeout/expiry path

### Step 8: validate and refine

Run:

```bash
quint parse model.qnt
quint typecheck model.qnt
quint run model.qnt --invariants criticalInvariant --max-steps=40
quint verify model.qnt --invariant criticalInvariant --max-steps=25
```

## Distillation matrix template

Use this worksheet during extraction.

| Source location | Trigger | Guard | State delta | Side effect | Quint action |
|-----------------|---------|-------|-------------|-------------|--------------|
| `auth/login.py:44` | login request | password valid and not locked | reset failed count | create session | `loginSuccess` |
| `auth/login.py:57` | login request | password invalid | increment count, maybe lock | lock email | `loginFailure` |
| `jobs/reset_sweep.ts:18` | scheduler tick | token expired | set token expired | none | `resetTokenExpires` |

## Mapping guide

| Implementation detail | Quint modeling approach |
|-----------------------|-------------------------|
| DB row collection | map/set state keyed by IDs |
| status column | sum type + map field |
| service method | action |
| if/guard clauses | action guards in `all { ... }` |
| scheduled task | time-guarded action |
| emitted event | event set/queue state update |
| external SDK side effect | imported module boundary + local reaction action |

## Pattern-specific distillation checklists

### Password auth with reset

- login success resets failed attempts
- login failure increments and triggers lockout threshold
- locked login attempts do not create sessions
- reset request allowed only for eligible statuses
- reset token expiry/consumption modeled explicitly

### RBAC

- role inheritance/effective permissions reflected
- membership add/remove/change actions extracted
- operation guards tied to can-read/can-write/can-admin
- owner invariants and self-removal policy modeled

### Resource invitations

- invitation status lifecycle complete
- existing user accept path modeled
- new user accept path modeled (if present)
- invite expiry and revocation logic preserved
- share permission propagation captured

### Soft delete

- delete sets metadata (who/when)
- restore window bounded by retention period
- retention expiry transitions to hard-delete/purged state
- bulk trash operations if present

### Notifications

- notification variants represented explicitly
- preference-dependent delivery behavior encoded
- digest batching and schedule transitions captured
- mark-read/archive transitions modeled

### Usage limits

- plan-tier limit tables captured
- quota checks on mutating operations preserved
- reset schedule logic modeled
- downgrade blocking semantics modeled

### Comments

- comment/reply/depth behavior captured
- mention extraction and mention notifications captured
- reply-author notification suppression rules captured
- reaction toggle/remove semantics captured

### OAuth/billing integrations

- external event hooks identified
- local policy reactions separated from provider mechanics
- subscription status transitions mapped from payment events
- suspension/cancellation edge cases represented

## Handling ambiguity and drift

When code is ambiguous:

1. Capture both plausible behaviors as comments.
2. Choose one model path with explicit assumption note.
3. Keep alternate as TODO action or branch if still live.

When code and tests disagree:

1. If tests are authoritative, model tested behavior.
2. If tests are stale, model runtime behavior and flag drift.

When multiple services diverge:

1. Model shared invariant target state.
2. Add source-specific action variants if behavior is intentionally different.

## Confidence tags

Annotate extracted actions with confidence in comments.

```quint
// confidence: high (backed by code + tests)
action createDocument(...) = ...

// confidence: medium (code path exists, limited test coverage)
action downgradeBlocked(...) = ...
```

## Distillation anti-patterns

- Copying endpoint/transport details into model state.
- Modeling unbounded IDs/counts before first verification pass.
- Ignoring rejected/blocked paths.
- Treating provider SDK behavior as local domain behavior.
- Ending without invariants and scenario runs.

## Done criteria

Distillation is done when:

1. Core lifecycles are explicit and complete.
2. Critical guards are represented.
3. Temporal behavior is represented.
4. Integration boundaries are explicit.
5. Invariants and runs detect regressions.
6. Assumptions are documented.

## Related workflows

- Use `elicit` for requirement-first modeling.
- Use `tend` for focused model edits.
- Use `weed` for ongoing model-vs-code drift checks.

## References

- `../../references/language-reference.md`
- `../../references/patterns.md`
- `../../references/test-generation.md`
- `references/worked-examples.md`
