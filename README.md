# Quint

*Velocity through executable clarity*

---

A skill package for building, distilling, and reviewing Quint (`.qnt`) specifications. [quint-lang.org](https://quint-lang.org/)

## Get started

Install Quint CLI:

```bash
npm i -g @informalsystems/quint
```

Install this skill package (example):

```bash
npx skills add juxt/quint
```

Then use:

- `/quint` for direct modeling work
- `/quint:elicit` for requirement-driven modeling
- `/quint:distill` for code-to-model extraction

## Working with agents

- [rule](.claude/rules/quint.md): auto-loads for `.qnt` files
- [tend](.claude/agents/tend.md): targeted model evolution
- [weed](.claude/agents/weed.md): spec-vs-code drift checks

## Why this package

- Keep behavior explicit as state transitions (`init`, domain actions, `step`).
- Add invariants and temporal checks before implementation drifts.
- Use simulations, model checking, and generated traces in one loop.
- Preserve intent across sessions and across engineers.

## What Quint captures (functional parity examples)

### Password reset flow

```quint
action requestPasswordReset(user: str, now: int) = all {
  Set(Active, Locked).contains(status.get(user)),
  resetTokenStatus' = resetTokenStatus.set(user, Pending),
  resetTokenExpiry' = resetTokenExpiry.set(user, now + RESET_TOKEN_EXPIRY),
  status' = status,
  failedAttempts' = failedAttempts,
  lockedUntil' = lockedUntil,
}

action completePasswordReset(user: str, now: int) = all {
  resetTokenStatus.get(user) == Pending,
  resetTokenExpiry.get(user) > now,
  resetTokenStatus' = resetTokenStatus.set(user, Used),
  failedAttempts' = failedAttempts.set(user, 0),
  status' = if (status.get(user) == Locked) status.set(user, Active) else status,
  lockedUntil' = if (status.get(user) == Locked) lockedUntil.set(user, 0) else lockedUntil,
}
```

This preserves the original behavior: active/locked users can request reset, pending tokens expire, successful reset clears lockout state.

### Circuit breaker behavior

```quint
type Breaker = | Closed | Open | HalfOpen

action recordFailure(service: str) = all {
  failures' = failures.setBy(service, n => n + 1),
  status' = if (failures.get(service) + 1 >= FAILURE_THRESHOLD)
    status.set(service, Open)
    else status,
}

action probeAfterTimeout(service: str, now: int) = all {
  status.get(service) == Open,
  openedAt.get(service) + RECOVERY_TIMEOUT <= now,
  status' = status.set(service, HalfOpen),
  failures' = failures,
}

action probeSuccess(service: str) = all {
  status.get(service) == HalfOpen,
  status' = status.set(service, Closed),
  failures' = failures.set(service, 0),
}
```

Same intent: failures trip the breaker, timeout allows probe, probe success closes the breaker.

### Incident escalation policy

```quint
action incidentEscalates(incident: int, now: int) = all {
  Set(Open, Investigating).contains(incidentStatus.get(incident)),
  declaredAt.get(incident) + slaTarget.get(incident) <= now,
  escalationLevel' = escalationLevel.setBy(incident, n => n + 1),
  execBriefingSent' = if (escalationLevel.get(incident) + 1 >= EXEC_NOTIFY_THRESHOLD)
    execBriefingSent.set(incident, true)
    else execBriefingSent,
  incidentStatus' = incidentStatus,
}
```

Same semantics: missed SLA escalates and eventually pages/briefs executive escalation tiers.

### Resource invitation lifecycle

```quint
action inviteToResource(invId: int, resource: str, email: str, permission: SharePermission, now: int) = all {
  invites.get(invId).status != Pending,
  invites' = invites.set(invId, {
    resource: resource,
    email: email,
    permission: permission,
    invitedBy: "actor",
    expiresAt: now + INVITATION_EXPIRY,
    status: Pending,
  }),
}

action invitationExpires(invId: int, now: int) = all {
  invites.get(invId).status == Pending,
  invites.get(invId).expiresAt <= now,
  invites' = invites.setBy(invId, i => { resource: i.resource, email: i.email, permission: i.permission, invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Expired }),
}
```

Same behavior: tokenized invitation is pending, accepted/declined/revoked, and expires by deadline.

### Usage limits and quotas

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

Same behavior: requests are bounded by plan; reset windows restore quota.

## What this looks like in practice

### Model catches unsafe shortcut

A request arrives to let suspended users reset passwords directly.

The model shows reset is guarded by `Set(Active, Locked)`, and account reinstatement is separate. That surfaces a policy question before code changes introduce a bypass.

### Cross-session knowledge persists

Days later, another engineer adds free trials. The model still enforces subscription/invoice invariants, surfacing whether zero-value invoices are required and whether payment method capture is mandatory before trial start.

### Access-control consequences become explicit

A team asks for “admins can view all billing history”. The RBAC model makes scope expansion explicit and highlights whether support needs read-only billing scopes instead of full billing write access.

### Drift checks are concrete

`weed` can report:

- model: lockout threshold is 5 attempts
- code: lockout threshold hardcoded to 3

You decide whether model or code is authoritative, then update exactly one side.

## Pattern catalog parity

The repository keeps the original functional pattern set, now in Quint:

1. Password Authentication with Reset
2. RBAC
3. Invitation to Resource
4. Soft Delete & Restore
5. Notification Preferences & Digests
6. Usage Limits & Quotas
7. Comments with Mentions
8. Integrating Library Specs (OAuth + billing)

See [patterns](references/patterns.md).

## Typical workflow

1. Model scope and state variables.
2. Add guarded actions and `step` composition.
3. Add invariants (`val`) and temporal checks (`temporal`).
4. Add `run` scenarios for happy path and failures.
5. Run `quint parse`, `quint typecheck`, `quint run`, and `quint verify`.

## Design review process

Language/pattern changes are reviewed through the panel workflow in [TEAM.md](TEAM.md):

- [REVIEW.md](REVIEW.md) for rough-edge fixes
- [PROPOSE.md](PROPOSE.md) for major additions
