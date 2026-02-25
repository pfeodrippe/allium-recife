# Complete patterns

This library contains reusable P patterns for common product and distributed-system behaviors.

Each pattern includes:

- behavior intent
- concrete P model sketch
- monitor/property examples
- test scenario guidance

## Pattern 1: Password Authentication with Reset

### Behavior

- login validates credentials
- repeated failures lock account
- reset request issues one-time token
- reset completion updates credential and revokes sessions

### P model example

```p
type tLoginReq = (reqId: int, userId: int, passwordHash: int, client: machine);
type tResetReq = (reqId: int, userId: int, client: machine);

event eLoginReq: tLoginReq;
event eLoginResp: (reqId: int, ok: bool);

event eResetReq: tResetReq;
event eResetIssued: (reqId: int, tokenId: int, ok: bool);
event eResetComplete: (reqId: int, tokenId: int, newHash: int, client: machine);

machine AuthService {
  var failCount: map[int, int];
  var locked: set[int];
  var tokenOwner: map[int, int];
  var usedToken: set[int];

  start state Ready {
    on eLoginReq do (r: tLoginReq) {
      if (r.userId in locked) {
        send r.client, eLoginResp, (reqId = r.reqId, ok = false);
      } else {
        send r.client, eLoginResp, (reqId = r.reqId, ok = true);
      }
    }

    on eResetReq do (r: tResetReq) {
      var tokenId: int;
      tokenId = r.reqId + 1000;
      tokenOwner[tokenId] = r.userId;
      send r.client, eResetIssued, (reqId = r.reqId, tokenId = tokenId, ok = true);
    }

    on eResetComplete do (r: (reqId: int, tokenId: int, newHash: int, client: machine)) {
      assert r.tokenId in tokenOwner, "unknown token";
      assert !(r.tokenId in usedToken), "token reuse";
      usedToken += (r.tokenId);
      send r.client, eLoginResp, (reqId = r.reqId, ok = true);
    }
  }
}

spec ResetTokenNotReused observes eResetComplete {
  var seen: set[int];

  start state S {
    on eResetComplete do (r: (reqId: int, tokenId: int, newHash: int, client: machine)) {
      assert !(r.tokenId in seen), "reset token reused";
      seen += (r.tokenId);
    }
  }
}
```

### Test focus

- duplicate completion with same token
- reset completion after expiry signal
- login attempts while locked

## Pattern 2: Role-Based Access Control (RBAC)

### Behavior

- principals own role sets
- roles imply permissions
- authorization decision is event-based and auditable

### P model example

```p
event eGrantRole: (principalId: int, roleId: int);
event eRevokeRole: (principalId: int, roleId: int);
event eAuthorize: (reqId: int, principalId: int, permissionId: int, client: machine);
event eAuthorizeResp: (reqId: int, allowed: bool);

machine Authorizer {
  var rolesByPrincipal: map[int, set[int]];
  var permissionsByRole: map[int, set[int]];

  start state Ready {
    on eGrantRole do (g: (principalId: int, roleId: int)) {
      if (!(g.principalId in rolesByPrincipal)) {
        rolesByPrincipal[g.principalId] = default(set[int]);
      }
      rolesByPrincipal[g.principalId] += (g.roleId);
    }

    on eRevokeRole do (g: (principalId: int, roleId: int)) {
      if (g.principalId in rolesByPrincipal) {
        rolesByPrincipal[g.principalId] -= (g.roleId);
      }
    }

    on eAuthorize do (a: (reqId: int, principalId: int, permissionId: int, client: machine)) {
      var allowed: bool;
      allowed = false;

      if (a.principalId in rolesByPrincipal) {
        // helper encapsulates role-to-permission traversal
        allowed = HasPermission(rolesByPrincipal[a.principalId], a.permissionId, permissionsByRole);
      }

      send a.client, eAuthorizeResp, (reqId = a.reqId, allowed = allowed);
    }
  }
}

fun HasPermission(roles: set[int], permissionId: int, permissionsByRole: map[int, set[int]]): bool;
```

### Property examples

- deny by default when principal has no roles
- revoke must affect subsequent authorization decisions

## Pattern 3: Invitation to Resource

### Behavior

- invitation lifecycle: pending -> accepted/declined/expired
- only accepted invitation grants membership

### P model example

```p
enum tInviteState { PENDING, ACCEPTED, DECLINED, EXPIRED }

event eInviteIssued: (inviteId: int, resourceId: int, inviteeId: int);
event eInviteAccept: (inviteId: int);
event eInviteDecline: (inviteId: int);
event eInviteExpire: (inviteId: int);
event eMembershipGranted: (resourceId: int, userId: int);

machine InvitationService {
  var stateByInvite: map[int, tInviteState];
  var resourceByInvite: map[int, int];
  var inviteeByInvite: map[int, int];

  start state Ready {
    on eInviteIssued do (x: (inviteId: int, resourceId: int, inviteeId: int)) {
      stateByInvite[x.inviteId] = PENDING;
      resourceByInvite[x.inviteId] = x.resourceId;
      inviteeByInvite[x.inviteId] = x.inviteeId;
    }

    on eInviteAccept do (x: (inviteId: int)) {
      assert stateByInvite[x.inviteId] == PENDING, "accept requires pending";
      stateByInvite[x.inviteId] = ACCEPTED;
      announce eMembershipGranted, (resourceId = resourceByInvite[x.inviteId], userId = inviteeByInvite[x.inviteId]);
    }

    on eInviteDecline do (x: (inviteId: int)) {
      assert stateByInvite[x.inviteId] == PENDING, "decline requires pending";
      stateByInvite[x.inviteId] = DECLINED;
    }

    on eInviteExpire do (x: (inviteId: int)) {
      if (stateByInvite[x.inviteId] == PENDING) {
        stateByInvite[x.inviteId] = EXPIRED;
      }
    }
  }
}
```

### Test focus

- accept vs expiry race
- duplicate accept
- decline after accept rejection

## Pattern 4: Soft Delete & Restore

### Behavior

- entities transition active/deleted
- active queries exclude deleted
- restore legal only from deleted state

### P model example

```p
enum tItemState { ACTIVE, DELETED }

event eDeleteReq: (itemId: int);
event eRestoreReq: (itemId: int);
event eListReq: (reqId: int, client: machine);
event eListResp: (reqId: int, activeItems: seq[int]);

machine ItemStore {
  var stateByItem: map[int, tItemState];

  start state Ready {
    on eDeleteReq do (r: (itemId: int)) {
      stateByItem[r.itemId] = DELETED;
    }

    on eRestoreReq do (r: (itemId: int)) {
      assert stateByItem[r.itemId] == DELETED, "restore requires deleted";
      stateByItem[r.itemId] = ACTIVE;
    }

    on eListReq do (r: (reqId: int, client: machine)) {
      var out: seq[int];
      out = ActiveItems(stateByItem);
      send r.client, eListResp, (reqId = r.reqId, activeItems = out);
    }
  }
}

fun ActiveItems(stateByItem: map[int, tItemState>): seq[int];
```

### Property examples

- deleted items never appear in active list response
- restore from active must fail/assert

## Pattern 5: Notification Preferences & Digests

### Behavior

- per-user channel enablement
- immediate notifications for enabled channels
- digest batching on scheduled tick

### P model example

```p
event ePreferenceSet: (userId: int, channelId: int, enabled: bool);
event eDomainEvent: (userId: int, eventId: int);
event eDigestTick;
event eNotificationSent: (userId: int, channelId: int, batchSize: int);

machine NotificationPlanner {
  var enabledByUser: map[int, set[int]];
  var pendingByUser: map[int, seq[int]];

  start state Ready {
    on ePreferenceSet do (p: (userId: int, channelId: int, enabled: bool)) {
      if (!(p.userId in enabledByUser)) {
        enabledByUser[p.userId] = default(set[int]);
      }
      if (p.enabled) {
        enabledByUser[p.userId] += (p.channelId);
      } else {
        enabledByUser[p.userId] -= (p.channelId);
      }
    }

    on eDomainEvent do (d: (userId: int, eventId: int)) {
      if (!(d.userId in pendingByUser)) {
        pendingByUser[d.userId] = default(seq[int]);
      }
      pendingByUser[d.userId] += (sizeof(pendingByUser[d.userId]), d.eventId);
    }

    on eDigestTick do {
      var userIds: seq[int];
      var i: int;
      var userId: int;
      userIds = PendingUserIds(pendingByUser);
      i = 0;
      while (i < sizeof(userIds)) {
        userId = userIds[i];
        announce eNotificationSent,
          (userId = userId, channelId = 1, batchSize = sizeof(pendingByUser[userId]));
        pendingByUser[userId] = default(seq[int]);
        i = i + 1;
      }
    }
  }
}

fun PendingUserIds(pendingByUser: map[int, seq[int]]): seq[int];
```

### Test focus

- preference changes near tick
- duplicate domain event handling
- disabled channels during pending queue

## Pattern 6: Usage Limits & Quotas

### Behavior

- consume request decrements quota when allowed
- denied consume preserves state
- reset restores allowance

### P model example

```p
event eConsumeReq: (reqId: int, accountId: int, units: int, client: machine);
event eConsumeResp: (reqId: int, allowed: bool, remaining: int);
event eQuotaReset: (accountId: int, maxUnits: int);

machine QuotaService {
  var remainingByAccount: map[int, int];

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

    on eQuotaReset do (x: (accountId: int, maxUnits: int)) {
      remainingByAccount[x.accountId] = x.maxUnits;
    }
  }
}
```

### Property examples

- remaining never negative
- denied consume does not mutate remaining

## Pattern 7: Comments with Mentions

### Behavior

- create/edit/delete comment workflow
- mention parsing triggers notifications
- permission checks gate edit/delete

### P model example

```p
event eCommentCreate: (commentId: int, authorId: int, mentionIds: set[int]);
event eCommentEdit: (commentId: int, editorId: int);
event eCommentDelete: (commentId: int, actorId: int);
event eMentionNotify: (commentId: int, targetId: int);

machine CommentService {
  var authorByComment: map[int, int];
  var deleted: set[int];

  start state Ready {
    on eCommentCreate do (c: (commentId: int, authorId: int, mentionIds: set[int])) {
      var mentionTargets: seq[int];
      var i: int;
      var uid: int;
      authorByComment[c.commentId] = c.authorId;
      mentionTargets = MentionTargets(c.mentionIds);
      i = 0;
      while (i < sizeof(mentionTargets)) {
        uid = mentionTargets[i];
        announce eMentionNotify, (commentId = c.commentId, targetId = uid);
        i = i + 1;
      }
    }

    on eCommentEdit do (e: (commentId: int, editorId: int)) {
      assert authorByComment[e.commentId] == e.editorId, "only author edits";
      assert !(e.commentId in deleted), "cannot edit deleted";
    }

    on eCommentDelete do (d: (commentId: int, actorId: int)) {
      assert authorByComment[d.commentId] == d.actorId, "only author deletes";
      deleted += (d.commentId);
    }
  }
}

fun MentionTargets(mentionIds: set[int]): seq[int];
```

### Test focus

- delete/edit race
- invalid editor path
- mention resolution on deleted comments

## Pattern 8: Integrating Library Specs

Split reusable protocol mechanics from app-domain policy using modules.

### Example: OAuth Authentication

Reusable auth module owns:

- challenge/callback protocol
- token/session lifecycle

Application module owns:

- user creation and role assignment
- suspension/reactivation policy

### Example: Payment Processing

Reusable payment module owns:

- intent/confirmation/failure protocol
- idempotency and retry semantics

Application module owns:

- subscription state changes
- entitlement and cancellation rules

### Library Spec Design Principles

- stable event protocol
- explicit extension points
- reusable monitor obligations
- avoid app-domain coupling inside integration module

## Using These Patterns

### Composition

Compose patterns through modules:

```p
module Auth = { AuthService };
module Billing = { QuotaService, PaymentCoordinator };
module Collaboration = { CommentService };

module Product = (union Auth, Billing, Collaboration);
module ProductChecked = assert ProductSafety, ProductLiveness in Product;
```

### Adaptation

Rename events and payload fields to domain vocabulary while preserving invariants and liveness obligations.

### Anti-Patterns

- copying model snippets without monitors/tests
- collapsing reusable boundaries into one monolithic machine
- hiding request/response IDs in concurrent protocols
- encoding only happy path and ignoring timeout/retry behavior
