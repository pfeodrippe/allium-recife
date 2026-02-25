# Worked examples: from code to P model

These examples show how to extract executable P models from existing implementation code.

Each example follows the same flow:

1. identify protocol events
2. recover state machines
3. recover invariants/liveness obligations
4. generate testcases and checker plan

## Example 1: Password Reset (Python/Flask)

### Implementation sketch

```python
# request reset
@app.post('/auth/request-reset')
def request_reset(email):
    user = User.find_by_email(email)
    if not user:
        return ok_generic()

    if user.status == 'deactivated':
        return ok_generic()

    Token.invalidate_pending(user.id)
    t = Token.issue(user.id, expires_in_hours=1)
    Mailer.send_reset(user.email, token=t.value)
    return ok_generic()

# complete reset
@app.post('/auth/reset')
def reset(token_value, new_password):
    if len(new_password) < 12:
        return error()

    t = Token.find(token_value)
    if not t or t.used or t.expired:
        return error()

    user = User.get(t.user_id)
    t.used = True
    user.password_hash = hash(new_password)
    user.failed_attempts = 0
    user.status = 'active'
    Session.revoke_all(user.id)
    Mailer.send_changed(user.email)
    return ok()
```

### Distillation notes

Behavioral details to keep:

- reset request is idempotent from an external observer standpoint
- pending tokens are invalidated before issuing a new one
- reset token is one-time and expiry-bounded
- successful reset revokes active sessions

Implementation details to drop:

- HTTP framework specifics
- hashing implementation
- ORM call shape

### Extracted P model

```p
type tResetReq = (reqId: int, userId: int, client: machine);
type tResetIssueResp = (reqId: int, ok: bool, tokenId: int);
type tResetCompleteReq = (reqId: int, tokenId: int, newHash: int, client: machine);
type tResetCompleteResp = (reqId: int, ok: bool);

enum tUserStatus { ACTIVE, LOCKED, DEACTIVATED }
enum tTokenStatus { PENDING, USED, EXPIRED }

event eResetRequest: tResetReq;
event eResetIssued: tResetIssueResp;
event eResetComplete: tResetCompleteReq;
event eResetCompleteResp: tResetCompleteResp;
event eTokenExpireTick;
event eSessionsRevoked: (userId: int);

machine AuthReset {
  var userStatus: map[int, tUserStatus];
  var tokenOwner: map[int, int];
  var tokenStatus: map[int, tTokenStatus];
  var pendingTokensByUser: map[int, set[int]];

  start state Ready {
    on eResetRequest do (r: tResetReq) {
      var tokenId: int;
      var existing: set[int];
      var seqIds: seq[int];
      var i: int;
      var t: int;

      if (userStatus[r.userId] == DEACTIVATED) {
        send r.client, eResetIssued, (reqId = r.reqId, ok = true, tokenId = 0);
        return;
      }

      if (!(r.userId in pendingTokensByUser)) {
        pendingTokensByUser[r.userId] = default(set[int]);
      }

      existing = pendingTokensByUser[r.userId];
      seqIds = SetToSeq(existing);
      i = 0;
      while (i < sizeof(seqIds)) {
        t = seqIds[i];
        tokenStatus[t] = USED;
        i = i + 1;
      }
      pendingTokensByUser[r.userId] = default(set[int]);

      tokenId = r.reqId + 100000;
      tokenOwner[tokenId] = r.userId;
      tokenStatus[tokenId] = PENDING;
      pendingTokensByUser[r.userId] += (tokenId);

      send r.client, eResetIssued, (reqId = r.reqId, ok = true, tokenId = tokenId);
    }

    on eResetComplete do (r: tResetCompleteReq) {
      var uid: int;

      if (!(r.tokenId in tokenOwner)) {
        send r.client, eResetCompleteResp, (reqId = r.reqId, ok = false);
        return;
      }

      if (tokenStatus[r.tokenId] != PENDING) {
        send r.client, eResetCompleteResp, (reqId = r.reqId, ok = false);
        return;
      }

      uid = tokenOwner[r.tokenId];
      tokenStatus[r.tokenId] = USED;
      userStatus[uid] = ACTIVE;
      announce eSessionsRevoked, (userId = uid);
      send r.client, eResetCompleteResp, (reqId = r.reqId, ok = true);
    }

    on eTokenExpireTick do {
      // elided: convert stale pending tokens to EXPIRED
    }
  }
}

fun SetToSeq(s: set[int]): seq[int];
```

### Extracted monitors

```p
spec ResetTokenSingleUse observes eResetComplete {
  var consumed: set[int];

  start state S {
    on eResetComplete do (r: tResetCompleteReq) {
      assert !(r.tokenId in consumed), "token reused";
      consumed += (r.tokenId);
    }
  }
}

spec ResetCompletionEventuallyResponds observes eResetComplete, eResetCompleteResp {
  var pending: set[int];

  start state Idle {
    on eResetComplete goto Waiting with (r: tResetCompleteReq) {
      pending += (r.reqId);
    }
  }

  hot state Waiting {
    on eResetComplete goto Waiting with (r: tResetCompleteReq) {
      pending += (r.reqId);
    }

    on eResetCompleteResp do (r: tResetCompleteResp) {
      assert r.reqId in pending, "response without matching request";
      pending -= (r.reqId);
      if (sizeof(pending) == 0) goto Idle;
    }
  }
}
```

### Extracted tests

```p
module AuthResetCore = { AuthReset, TestDriver };
module AuthResetChecked = assert ResetTokenSingleUse, ResetCompletionEventuallyResponds in AuthResetCore;

test tcResetNominal [main=TestDriver]:
  AuthResetChecked;

test tcResetDuplicateComplete [main=DuplicateCompleteDriver]:
  AuthResetChecked;
```

Recommended checker runs:

- smoke: `p check -tc tcResetNominal -s 1`
- local: `p check -tc tcResetNominal -s 100`
- adversarial: `p check -tc tcResetDuplicateComplete -s 1000`

## Example 2: Usage Limits (TypeScript/Node)

### Implementation sketch

```ts
async function consume(accountId: string, units: number, reqId: string) {
  const q = await quotaStore.read(accountId);
  if (units <= q.remaining) {
    await quotaStore.write(accountId, q.remaining - units);
    return { reqId, allowed: true, remaining: q.remaining - units };
  }
  return { reqId, allowed: false, remaining: q.remaining };
}

cron.everyHour(async () => {
  for (const accountId of await quotaStore.allAccounts()) {
    await quotaStore.resetWindow(accountId);
  }
});
```

### Distillation notes

Keep:

- consume request/response protocol
- denial does not mutate state
- periodic reset event

Drop:

- persistence details
- cron framework API

### Extracted P model

```p
event eConsumeReq: (reqId: int, accountId: int, units: int, client: machine);
event eConsumeResp: (reqId: int, allowed: bool, remaining: int);
event eQuotaResetTick;

machine QuotaService {
  var remainingByAccount: map[int, int];
  var defaultLimitByAccount: map[int, int];

  start state Ready {
    on eConsumeReq do (r: (reqId: int, accountId: int, units: int, client: machine)) {
      var rem: int;
      rem = remainingByAccount[r.accountId];

      if (r.units <= rem) {
        remainingByAccount[r.accountId] = rem - r.units;
        send r.client, eConsumeResp,
          (reqId = r.reqId, allowed = true, remaining = remainingByAccount[r.accountId]);
      } else {
        send r.client, eConsumeResp,
          (reqId = r.reqId, allowed = false, remaining = rem);
      }
    }

    on eQuotaResetTick do {
      // elided: restore account remaining to defaultLimitByAccount values
    }
  }
}
```

### Extracted monitors

```p
spec RemainingNeverNegative observes eConsumeResp {
  start state S {
    on eConsumeResp do (r: (reqId: int, allowed: bool, remaining: int)) {
      assert r.remaining >= 0, "remaining below zero";
    }
  }
}

spec EveryConsumeResponds observes eConsumeReq, eConsumeResp {
  var pending: set[int];

  start state Idle {
    on eConsumeReq goto Waiting with (r: (reqId: int, accountId: int, units: int, client: machine)) {
      pending += (r.reqId);
    }
  }

  hot state Waiting {
    on eConsumeReq goto Waiting with (r: (reqId: int, accountId: int, units: int, client: machine)) {
      pending += (r.reqId);
    }

    on eConsumeResp do (r: (reqId: int, allowed: bool, remaining: int)) {
      assert r.reqId in pending, "response without request";
      pending -= (r.reqId);
      if (sizeof(pending) == 0) goto Idle;
    }
  }
}
```

### Extracted tests

```p
module QuotaCore = { QuotaService, QuotaDriver };
module QuotaChecked = assert RemainingNeverNegative, EveryConsumeResponds in QuotaCore;

test tcQuotaNormal [main=QuotaDriver]:
  QuotaChecked;

param nBurst: int;

test param (nBurst in [1, 2, 4, 8]) tcQuotaBurst [main=QuotaBurstDriver]:
  QuotaChecked;
```

## Example 3: Soft Delete (Java/Spring)

### Implementation sketch

```java
public void softDelete(long itemId) {
  Item i = repo.find(itemId);
  i.state = DELETED;
  repo.save(i);
}

public void restore(long itemId) {
  Item i = repo.find(itemId);
  if (i.state != DELETED) throw new InvalidState();
  i.state = ACTIVE;
  repo.save(i);
}

public List<Item> listActive() {
  return repo.findByState(ACTIVE);
}
```

### Distillation notes

Keep:

- state transition rules
- list semantics excluding deleted items

Drop:

- repository method names and transaction annotations

### Extracted P model

```p
enum tItemState { ACTIVE, DELETED }

event eDeleteReq: (reqId: int, itemId: int, client: machine);
event eRestoreReq: (reqId: int, itemId: int, client: machine);
event eListActiveReq: (reqId: int, client: machine);
event eListActiveResp: (reqId: int, activeItems: seq[int]);

machine ItemLifecycle {
  var stateByItem: map[int, tItemState];

  start state Ready {
    on eDeleteReq do (r: (reqId: int, itemId: int, client: machine)) {
      stateByItem[r.itemId] = DELETED;
    }

    on eRestoreReq do (r: (reqId: int, itemId: int, client: machine)) {
      assert stateByItem[r.itemId] == DELETED, "restore requires deleted";
      stateByItem[r.itemId] = ACTIVE;
    }

    on eListActiveReq do (r: (reqId: int, client: machine)) {
      send r.client, eListActiveResp,
        (reqId = r.reqId, activeItems = ActiveItems(stateByItem));
    }
  }
}

fun ActiveItems(stateByItem: map[int, tItemState]): seq[int];
```

### Extracted monitor

```p
spec DeletedItemsNotListed observes eDeleteReq, eListActiveResp {
  var deleted: set[int];

  start state S {
    on eDeleteReq do (r: (reqId: int, itemId: int, client: machine)) {
      deleted += (r.itemId);
    }

    on eListActiveResp do (r: (reqId: int, activeItems: seq[int])) {
      var i: int;
      i = 0;
      while (i < sizeof(r.activeItems)) {
        assert !(r.activeItems[i] in deleted), "deleted item appears in active list";
        i = i + 1;
      }
    }
  }
}
```

### Extracted tests

```p
module ItemCore = { ItemLifecycle, ItemDriver };
module ItemChecked = assert DeletedItemsNotListed in ItemCore;

test tcDeleteList [main=ItemDriver]:
  ItemChecked;

test tcRestoreStateGuard [main=RestoreGuardDriver]:
  ItemChecked;
```

## Distillation checklist from examples

Before finalizing any extraction:

- protocol events are explicit and typed
- machine states/transitions are explicit
- safety and liveness properties are monitor-backed
- tests include at least one adversarial case
- dropped implementation details are documented intentionally
