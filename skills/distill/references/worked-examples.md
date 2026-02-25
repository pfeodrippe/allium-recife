# Worked distillation examples

These examples preserve the same functional scenario set as earlier materials, now distilled into Quint.

Each example shows:

1. Source implementation excerpt.
2. Distilled Quint model fragment.
3. Mapping notes from code to model.

## Example 1: Password reset + lockout

### Source implementation (Python excerpt)

```python
MAX_LOGIN_ATTEMPTS = 5
LOCKOUT_MINUTES = 15
RESET_TOKEN_EXPIRY_MINUTES = 60

def login(email, password):
    user = User.find_by_email(email)
    if not user:
        return None

    if user.status == "locked" and user.locked_until > now():
        raise AccountLocked(user.locked_until)

    if verify(password, user.password_hash):
        user.failed_login_attempts = 0
        return Session.create(user_id=user.id, expires_at=now() + hours(24))

    user.failed_login_attempts += 1
    if user.failed_login_attempts >= MAX_LOGIN_ATTEMPTS:
        user.status = "locked"
        user.locked_until = now() + minutes(LOCKOUT_MINUTES)
    user.save()


def request_password_reset(email):
    user = User.find_by_email(email)
    if not user or user.status not in ["active", "locked"]:
        return

    PasswordResetToken.invalidate_pending(user.id)
    token = PasswordResetToken.create(
        user_id=user.id,
        expires_at=now() + minutes(RESET_TOKEN_EXPIRY_MINUTES),
        status="pending"
    )
    Email.send_password_reset(user.email, token.value)
```

### Distilled Quint model

```quint
module PasswordResetDistilled {
  const USERS: Set[str]
  const NOW_VALUES: Set[int]

  const MAX_LOGIN_ATTEMPTS: int
  const LOCKOUT_DURATION: int
  const RESET_TOKEN_EXPIRY: int

  type UserStatus = | Active | Locked | Deactivated
  type TokenStatus = | NoToken | Pending | Used | Expired

  var status: str -> UserStatus
  var failedAttempts: str -> int
  var lockedUntil: str -> int
  var sessionCount: str -> int
  var resetTokenStatus: str -> TokenStatus
  var resetTokenExpiry: str -> int

  action init = all {
    status' = USERS.mapBy(_ => Active),
    failedAttempts' = USERS.mapBy(_ => 0),
    lockedUntil' = USERS.mapBy(_ => 0),
    sessionCount' = USERS.mapBy(_ => 0),
    resetTokenStatus' = USERS.mapBy(_ => NoToken),
    resetTokenExpiry' = USERS.mapBy(_ => 0),
  }

  action loginSuccess(user: str, now: int) = all {
    status.get(user) == Active,
    not(status.get(user) == Locked and lockedUntil.get(user) > now),
    failedAttempts' = failedAttempts.set(user, 0),
    sessionCount' = sessionCount.setBy(user, n => n + 1),
    status' = status,
    lockedUntil' = lockedUntil,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiry' = resetTokenExpiry,
  }

  action loginFailure(user: str, now: int) = all {
    status.get(user) == Active,
    failedAttempts' = failedAttempts.setBy(user, n => n + 1),
    status' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
      status.set(user, Locked)
      else status,
    lockedUntil' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
      lockedUntil.set(user, now + LOCKOUT_DURATION)
      else lockedUntil,
    sessionCount' = sessionCount,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiry' = resetTokenExpiry,
  }

  action requestPasswordReset(user: str, now: int) = all {
    Set(Active, Locked).contains(status.get(user)),
    resetTokenStatus' = resetTokenStatus.set(user, Pending),
    resetTokenExpiry' = resetTokenExpiry.set(user, now + RESET_TOKEN_EXPIRY),
    status' = status,
    failedAttempts' = failedAttempts,
    lockedUntil' = lockedUntil,
    sessionCount' = sessionCount,
  }

  action resetTokenExpires(user: str, now: int) = all {
    resetTokenStatus.get(user) == Pending,
    resetTokenExpiry.get(user) <= now,
    resetTokenStatus' = resetTokenStatus.set(user, Expired),
    status' = status,
    failedAttempts' = failedAttempts,
    lockedUntil' = lockedUntil,
    sessionCount' = sessionCount,
    resetTokenExpiry' = resetTokenExpiry,
  }

  val attemptsNonNegative = USERS.forall(u => failedAttempts.get(u) >= 0)
}
```

### Mapping notes

- `failed_login_attempts` -> `failedAttempts` map.
- lockout threshold branch -> conditional `status'` / `lockedUntil'` updates.
- reset token invalidation + create -> modeled as status replacement to `Pending` with new expiry.

---

## Example 2: Usage limits and rate limiting

### Source implementation (TypeScript excerpt)

```ts
function createDocument(workspace: Workspace) {
  if (workspace.documentsCount >= workspace.plan.maxDocuments) {
    throw new LimitReached("documents")
  }
  workspace.documentsCount += 1
}

function recordApiRequest(workspace: Workspace) {
  if (workspace.apiRequestsToday >= workspace.plan.maxApiRequestsPerDay) {
    throw new RateLimited(workspace.nextResetAt)
  }
  workspace.apiRequestsToday += 1
}

function resetDailyUsage(workspace: Workspace, now: Date) {
  if (workspace.nextResetAt <= now) {
    workspace.apiRequestsToday = 0
    workspace.nextResetAt = addDays(workspace.nextResetAt, 1)
  }
}
```

### Distilled Quint model

```quint
module UsageLimitsDistilled {
  const WORKSPACES: Set[str]
  const NOW_VALUES: Set[int]

  type Plan = | Free | Pro | Enterprise

  var plan: str -> Plan
  var documents: str -> int
  var apiRequestsToday: str -> int
  var nextResetAt: str -> int

  pure def maxDocuments(p: Plan): int =
    match p { | Free => 10 | Pro => 1000 | Enterprise => 1_000_000 }

  pure def maxApiRequests(p: Plan): int =
    match p { | Free => 100 | Pro => 10_000 | Enterprise => 1_000_000 }

  action init = all {
    plan' = WORKSPACES.mapBy(_ => Free),
    documents' = WORKSPACES.mapBy(_ => 0),
    apiRequestsToday' = WORKSPACES.mapBy(_ => 0),
    nextResetAt' = WORKSPACES.mapBy(_ => 24),
  }

  action createDocument(w: str) = all {
    documents.get(w) < maxDocuments(plan.get(w)),
    documents' = documents.setBy(w, n => n + 1),
    plan' = plan,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
  }

  action createDocumentLimitReached(w: str) = all {
    documents.get(w) >= maxDocuments(plan.get(w)),
    documents' = documents,
    plan' = plan,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
  }

  action recordApiRequest(w: str) = all {
    apiRequestsToday.get(w) < maxApiRequests(plan.get(w)),
    apiRequestsToday' = apiRequestsToday.setBy(w, n => n + 1),
    documents' = documents,
    plan' = plan,
    nextResetAt' = nextResetAt,
  }

  action apiRateLimitExceeded(w: str) = all {
    apiRequestsToday.get(w) >= maxApiRequests(plan.get(w)),
    apiRequestsToday' = apiRequestsToday,
    documents' = documents,
    plan' = plan,
    nextResetAt' = nextResetAt,
  }

  action resetDailyUsage(w: str, now: int) = all {
    nextResetAt.get(w) <= now,
    apiRequestsToday' = apiRequestsToday.set(w, 0),
    nextResetAt' = nextResetAt.set(w, now + 24),
    documents' = documents,
    plan' = plan,
  }

  val docsNonNegative = WORKSPACES.forall(w => documents.get(w) >= 0)
}
```

### Mapping notes

- limit guard failures become explicit blocked actions.
- daily scheduler branch becomes time-guarded `resetDailyUsage`.
- plan-dependent limits captured as pure lookup functions.

---

## Example 3: Soft delete with retention expiry

### Source implementation (Java excerpt)

```java
public void deleteDocument(User actor, Document doc) {
  if (doc.status != Status.ACTIVE) throw new IllegalStateException();
  doc.status = Status.DELETED;
  doc.deletedAt = Instant.now();
  doc.deletedBy = actor.id;
}

public void restoreDocument(Document doc) {
  if (doc.status != Status.DELETED) throw new IllegalStateException();
  if (doc.deletedAt.plus(retention).isBefore(Instant.now())) throw new IllegalStateException();
  doc.status = Status.ACTIVE;
  doc.deletedAt = null;
  doc.deletedBy = null;
}

public void retentionSweep(Document doc) {
  if (doc.status == Status.DELETED && doc.deletedAt.plus(retention).isBefore(Instant.now())) {
    repository.hardDelete(doc.id);
  }
}
```

### Distilled Quint model

```quint
module SoftDeleteDistilled {
  const DOCS: Set[int]
  const USERS: Set[str]
  const NOW_VALUES: Set[int]
  const RETENTION_PERIOD: int

  type DocStatus = | Active | Deleted | Purged

  var status: int -> DocStatus
  var deletedAt: int -> int
  var deletedBy: int -> str

  action init = all {
    status' = DOCS.mapBy(_ => Active),
    deletedAt' = DOCS.mapBy(_ => 0),
    deletedBy' = DOCS.mapBy(_ => ""),
  }

  action deleteDocument(actor: str, doc: int, now: int) = all {
    USERS.contains(actor),
    status.get(doc) == Active,
    status' = status.set(doc, Deleted),
    deletedAt' = deletedAt.set(doc, now),
    deletedBy' = deletedBy.set(doc, actor),
  }

  action restoreDocument(actor: str, doc: int, now: int) = all {
    USERS.contains(actor),
    status.get(doc) == Deleted,
    deletedAt.get(doc) + RETENTION_PERIOD > now,
    status' = status.set(doc, Active),
    deletedAt' = deletedAt.set(doc, 0),
    deletedBy' = deletedBy.set(doc, ""),
  }

  action retentionSweep(doc: int, now: int) = all {
    status.get(doc) == Deleted,
    deletedAt.get(doc) + RETENTION_PERIOD <= now,
    status' = status.set(doc, Purged),
    deletedAt' = deletedAt,
    deletedBy' = deletedBy,
  }

  val noNegativeTimes = DOCS.forall(d => deletedAt.get(d) >= 0)
}
```

### Mapping notes

- hard-delete repository call becomes `Purged` terminal state.
- restore window encoded directly as guard on retention duration.

---

## Example 4: Resource invitation lifecycle

### Source implementation (Python excerpt)

```python
INVITATION_EXPIRY_DAYS = 7

def invite_to_resource(inviter, resource, email, permission):
    existing = Invitation.find_active(resource_id=resource.id, email=email)
    if existing:
        raise InvitationAlreadyPending()

    invite = Invitation.create(
        resource_id=resource.id,
        email=email,
        permission=permission,
        invited_by=inviter.id,
        expires_at=now() + days(INVITATION_EXPIRY_DAYS),
        status="pending"
    )
    Email.send_resource_invite(email, invite.token)
    return invite


def accept_invitation(invite, user):
    if invite.status != "pending" or invite.expires_at <= now():
        raise InvalidInvite()

    invite.status = "accepted"
    Share.create(resource_id=invite.resource_id, user_id=user.id, permission=invite.permission)


def expire_invitations(now_ts):
    Invitation.where(status="pending", expires_at__lte=now_ts).update(status="expired")
```

### Distilled Quint model

```quint
module ResourceInvitationDistilled {
  const INVITATIONS: Set[int]
  const RESOURCES: Set[str]
  const USERS: Set[str]
  const NOW_VALUES: Set[int]
  const INVITATION_EXPIRY: int

  type SharePermission = | View | Edit | Admin
  type InviteStatus = | Pending | Accepted | Declined | Expired | Revoked

  type Invite = {
    resource: str,
    email: str,
    permission: SharePermission,
    invitedBy: str,
    expiresAt: int,
    status: InviteStatus,
  }

  var invite: int -> Invite
  var sharePermission: (str, str) -> SharePermission
  var shareActive: (str, str) -> bool

  action init = all {
    invite' = INVITATIONS.mapBy(_ => {
      resource: "",
      email: "",
      permission: View,
      invitedBy: "",
      expiresAt: 0,
      status: Revoked,
    }),
    sharePermission' = RESOURCES.cross(USERS).mapBy(_ => View),
    shareActive' = RESOURCES.cross(USERS).mapBy(_ => false),
  }

  action inviteToResource(id: int, inviter: str, resource: str, email: str, permission: SharePermission, now: int) = all {
    invite.get(id).status != Pending,
    invite' = invite.set(id, {
      resource: resource,
      email: email,
      permission: permission,
      invitedBy: inviter,
      expiresAt: now + INVITATION_EXPIRY,
      status: Pending,
    }),
    sharePermission' = sharePermission,
    shareActive' = shareActive,
  }

  action acceptInvitation(id: int, user: str, now: int) = all {
    invite.get(id).status == Pending,
    invite.get(id).expiresAt > now,
    invite' = invite.setBy(id, i => {
      resource: i.resource,
      email: i.email,
      permission: i.permission,
      invitedBy: i.invitedBy,
      expiresAt: i.expiresAt,
      status: Accepted,
    }),
    sharePermission' = sharePermission.set((invite.get(id).resource, user), invite.get(id).permission),
    shareActive' = shareActive.set((invite.get(id).resource, user), true),
  }

  action expireInvitation(id: int, now: int) = all {
    invite.get(id).status == Pending,
    invite.get(id).expiresAt <= now,
    invite' = invite.setBy(id, i => {
      resource: i.resource,
      email: i.email,
      permission: i.permission,
      invitedBy: i.invitedBy,
      expiresAt: i.expiresAt,
      status: Expired,
    }),
    sharePermission' = sharePermission,
    shareActive' = shareActive,
  }
}
```

### Mapping notes

- unique pending invite constraint captured as `status != Pending` at creation point.
- scheduler update query becomes `expireInvitation` transition.

---

## Example 5: Comments, mentions, and notification triggers

### Source implementation (TypeScript excerpt)

```ts
function createReply(authorId: string, parentId: string, body: string) {
  const parent = getComment(parentId)
  if (parent.status !== "active") throw new Error("invalid_parent")

  const comment = insertComment({
    authorId,
    parentId,
    body,
    status: "active",
  })

  const mentions = parseMentions(body)
  mentions.forEach(userId => queueMentionNotification(userId, comment.id, authorId))

  if (parent.authorId !== authorId && !mentions.includes(parent.authorId)) {
    queueReplyNotification(parent.authorId, comment.id, parent.id)
  }

  return comment
}

function toggleReaction(commentId: string, userId: string, emoji: string) {
  const existing = findReaction(commentId, userId, emoji)
  if (existing) removeReaction(existing.id)
  else insertReaction({ commentId, userId, emoji })
}
```

### Distilled Quint model

```quint
module CommentsDistilled {
  const COMMENTS: Set[int]
  const USERS: Set[str]

  type CommentStatus = | Active | Deleted
  type Reaction = (int, str, str)

  var status: int -> CommentStatus
  var author: int -> str
  var replyTo: int -> int
  var mentionsByComment: int -> Set[str]
  var reactions: Set[Reaction]

  var mentionEvents: Set[(str, int, str)]
  var replyEvents: Set[(str, int, int)]

  action init = all {
    status' = COMMENTS.mapBy(_ => Active),
    author' = COMMENTS.mapBy(_ => ""),
    replyTo' = COMMENTS.mapBy(_ => 0),
    mentionsByComment' = COMMENTS.mapBy(_ => Set()),
    reactions' = Set(),
    mentionEvents' = Set(),
    replyEvents' = Set(),
  }

  action createReply(c: int, parent: int, u: str, mentionedUsers: Set[str]) = all {
    status.get(parent) == Active,
    author' = author.set(c, u),
    replyTo' = replyTo.set(c, parent),
    mentionsByComment' = mentionsByComment.set(c, mentionedUsers),
    mentionEvents' = mentionedUsers.map(mu => (mu, c, u)),
    replyEvents' = if (author.get(parent) != u and not(mentionedUsers.contains(author.get(parent))))
      replyEvents.union(Set((author.get(parent), c, parent)))
      else replyEvents,
    status' = status,
    reactions' = reactions,
  }

  action toggleReaction(c: int, u: str, emoji: str) = all {
    reactions' = if (reactions.contains((c, u, emoji)))
      reactions.exclude((c, u, emoji))
      else reactions.union(Set((c, u, emoji))),
    status' = status,
    author' = author,
    replyTo' = replyTo,
    mentionsByComment' = mentionsByComment,
    mentionEvents' = mentionEvents,
    replyEvents' = replyEvents,
  }
}
```

### Mapping notes

- `parseMentions` output becomes explicit action argument `mentionedUsers`.
- reply-notification suppression branch is preserved as guard in `replyEvents'` update.
- reaction upsert behavior becomes a toggle action on reaction set.

---

## Example 6: OAuth + billing integration boundaries

### Source implementation (mixed excerpt)

```python
# auth handler
identity = oauth.exchange(code)
user = User.find_or_create(email=identity.email)
if user.status == "suspended":
    revoke_session(identity.session_id)
    return error("suspended")
user.last_login_at = now()

# webhook handler
if event.type == "invoice.payment_failed":
    sub = Subscription.find_by_customer(event.customer)
    sub.status = "past_due"
elif event.type == "invoice.paid":
    sub = Subscription.find_by_customer(event.customer)
    sub.status = "active"
```

### Distilled Quint modules

```quint
module AppAuthDistilled {
  import OAuth.* from "oauth2"

  const USERS: Set[str]

  type UserStatus = | Active | Suspended | Deactivated

  var status: str -> UserStatus
  var lastLoginAt: str -> int

  action init = all {
    status' = USERS.mapBy(_ => Active),
    lastLoginAt' = USERS.mapBy(_ => 0),
  }

  action updateUserOnLogin(user: str, now: int) = all {
    status.get(user) == Active,
    lastLoginAt' = lastLoginAt.set(user, now),
    status' = status,
  }

  action blockSuspendedUserLogin(user: str) = all {
    status.get(user) == Suspended,
    status' = status,
    lastLoginAt' = lastLoginAt,
  }
}

module BillingDistilled {
  import StripeBilling.* from "stripe-billing"

  const ORGS: Set[str]

  type SubscriptionStatus = | Trialing | Active | PastDue | Cancelled | Expired

  var subStatus: str -> SubscriptionStatus

  action init = subStatus' = ORGS.mapBy(_ => Trialing)

  action paymentFailed(org: str) =
    subStatus' = subStatus.set(org, PastDue)

  action paymentSucceeded(org: str) =
    subStatus' = subStatus.set(org, Active)
}
```

### Mapping notes

- provider mechanics stay in imported modules; local module reacts to provider events.
- suspended-user policy stays local and explicit.
- payment webhooks map to subscription state transitions.

---

## Distillation notes

- Keep finite domains first; scale state space later.
- Preserve behavior semantics before optimizing model shape.
- Add blocked-path actions when code has explicit rejection branches.
- Add one run and one invariant immediately after first extraction.
