# Test generation

From a Quint specification, generate the same functional test categories as the previous workflow, with explicit support for the core scenario set.

## 1. Contract tests (per action)

For every action:

1. Enabled case: guards hold, expected next-state predicates hold.
2. Disabled cases: falsify one guard at a time and confirm action is not enabled.
3. Boundary cases: threshold edges (equal, just below, just above).

Template checklist:

- guard coverage complete
- all updated vars asserted
- unchanged vars asserted where relevant

## 2. Lifecycle transition tests (per status model)

For each lifecycle map:

- allowed transitions are reachable
- forbidden transitions are unreachable
- terminal states remain terminal unless explicit recovery action exists

Use coverage matrix:

| Entity lifecycle | Expected transitions | Forbidden transitions |
|------------------|----------------------|-----------------------|
| InviteStatus | Pending->Accepted/Declined/Expired/Revoked | Accepted->Pending |
| UserStatus | Active->Locked, Locked->Active | Deactivated->Active (if not supported) |
| SubscriptionStatus | Trialing->Active/PastDue, Active->Cancelled | Cancelled->Active (if forbidden) |

## 3. Temporal tests (per time-guarded action)

For each temporal guard `deadline <= now`:

- before deadline: disabled
- at deadline: enabled and correct transition
- after deadline: repeated runs preserve invariants (idempotent or terminal)

Targets include:

- lockout expiry
- invitation expiry
- reset token expiry
- daily quota reset
- retention purge
- trial reminder windows

## 4. Communication/event tests

When model includes event sets/outboxes:

- verify recipient/target identity
- verify event type/variant
- verify event suppression rules (no duplicate mention + reply)

## 5. Scenario tests (end-to-end runs)

Per feature area add at least:

1. happy-path run
2. blocked-path run
3. timeout/expiry run

Recommended scenario families:

- password lockout then reset
- invitation create -> accept -> share activation
- invitation create -> expire
- usage within limits then blocked at limit
- soft delete -> restore within retention
- soft delete -> purge after retention
- comment reply with mention suppression behavior
- payment failure then recovery

## 6. Variant and match tests

For sum types and pattern matches:

- each constructor appears in at least one run
- match expressions are branch-complete for used variants
- variant-specific fields are accessed only in valid branch contexts

## 7. Cross-action interaction tests

Generate interleavings for actions touching same state.

Examples:

- `loginFailure` vs `lockoutExpires`
- `acceptInvitation` vs `invitationExpires`
- `downgradePlan` vs `createDocument`
- `createReply` vs `deleteComment`

Assertions:

- invariants always hold
- no impossible state combinations emerge

## 8. Property-focused verification plan

For each critical invariant:

1. quick simulation pass (`run`)
2. bounded verify pass (`verify`)
3. backend comparison for high-risk models (Apalache and TLC when needed)

## Command templates

```bash
# Parse and typecheck
quint parse model.qnt
quint typecheck model.qnt

# Fast simulation-based invariant checks
quint run model.qnt --invariants attemptsNonNegative sessionsNonNegative --max-steps=60

# Model-based trace generation for implementation replay
quint run model.qnt --mbt --out-itf=traces/trace_{seq}.itf.json --n-traces=200 --max-steps=50

# Bounded verification of critical invariants
quint verify model.qnt --invariant attemptsNonNegative --max-steps=30
quint verify model.qnt --invariant inviteLifecycleValid --max-steps=30

# Alternate backend cross-check where required
quint verify model.qnt --backend=tlc --invariant attemptsNonNegative
```

## Minimal CI test policy

Per module in CI:

1. `parse` + `typecheck`
2. one fast `run` invariant suite
3. one `verify` invariant suite
4. optional MBT trace export for downstream tests

## Failure triage guide

When a check fails:

1. classify: model bug vs assumption gap vs intended behavior not yet modeled
2. isolate smallest failing run
3. add regression `run` scenario reproducing failure
4. update action guards/state updates or invariant definition
