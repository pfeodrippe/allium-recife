# Complete patterns

This library keeps the same functional pattern set as the original version, now expressed in Quint.

| Pattern | Key Features Demonstrated |
|---------|---------------------------|
| Password Authentication with Reset | Lockout, session lifecycle, reset token lifecycle, temporal checks |
| Role-Based Access Control (RBAC) | Role inheritance, permission checks, membership management |
| Invitation to Resource | Invitation lifecycle, guest/member flows, share permissions |
| Soft Delete & Restore | Deleted-state lifecycle, retention expiry, restore window |
| Notification Preferences & Digests | Notification variants, per-type prefs, digest batching |
| Usage Limits & Quotas | Plan limits, quota enforcement, reset windows, upgrades/downgrades |
| Comments with Mentions | Threading, mention parsing, mention/reply notifications, reactions |
| Integrating Library Specs | OAuth + billing composition with imported modules |

---

## Pattern 1: Password Authentication with Reset

**Demonstrates:** lockout policy, session expiry, password reset request/complete/expire flows

```quint
module PasswordAuth {
  const USERS: Set[str]
  const MAX_LOGIN_ATTEMPTS: int
  const LOCKOUT_DURATION: int
  const RESET_TOKEN_EXPIRY: int
  const NOW_VALUES: Set[int]

  type UserStatus = | Active | Locked | Deactivated
  type TokenStatus = | NoToken | Pending | Used | Expired

  var status: str -> UserStatus
  var failedAttempts: str -> int
  var lockedUntil: str -> int
  var activeSessions: str -> int
  var resetTokenStatus: str -> TokenStatus
  var resetTokenExpiresAt: str -> int

  action init = all {
    status' = USERS.mapBy(_ => Active),
    failedAttempts' = USERS.mapBy(_ => 0),
    lockedUntil' = USERS.mapBy(_ => 0),
    activeSessions' = USERS.mapBy(_ => 0),
    resetTokenStatus' = USERS.mapBy(_ => NoToken),
    resetTokenExpiresAt' = USERS.mapBy(_ => 0),
  }

  pure def isLocked(user: str, now: int): bool =
    status.get(user) == Locked and lockedUntil.get(user) > now

  action loginSuccess(user: str, now: int) = all {
    USERS.contains(user),
    status.get(user) == Active,
    not(isLocked(user, now)),
    failedAttempts' = failedAttempts.set(user, 0),
    activeSessions' = activeSessions.setBy(user, n => n + 1),
    status' = status,
    lockedUntil' = lockedUntil,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action loginFailure(user: str, now: int) = all {
    USERS.contains(user),
    status.get(user) == Active,
    not(isLocked(user, now)),
    failedAttempts' = failedAttempts.setBy(user, n => n + 1),
    status' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
      status.set(user, Locked)
      else status,
    lockedUntil' = if (failedAttempts.get(user) + 1 >= MAX_LOGIN_ATTEMPTS)
      lockedUntil.set(user, now + LOCKOUT_DURATION)
      else lockedUntil,
    activeSessions' = activeSessions,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action lockoutExpires(user: str, now: int) = all {
    USERS.contains(user),
    status.get(user) == Locked,
    lockedUntil.get(user) <= now,
    status' = status.set(user, Active),
    failedAttempts' = failedAttempts.set(user, 0),
    lockedUntil' = lockedUntil.set(user, 0),
    activeSessions' = activeSessions,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action logout(user: str) = all {
    USERS.contains(user),
    activeSessions.get(user) > 0,
    activeSessions' = activeSessions.setBy(user, n => n - 1),
    status' = status,
    failedAttempts' = failedAttempts,
    lockedUntil' = lockedUntil,
    resetTokenStatus' = resetTokenStatus,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action requestPasswordReset(user: str, now: int) = all {
    USERS.contains(user),
    Set(Active, Locked).contains(status.get(user)),
    resetTokenStatus' = resetTokenStatus.set(user, Pending),
    resetTokenExpiresAt' = resetTokenExpiresAt.set(user, now + RESET_TOKEN_EXPIRY),
    status' = status,
    failedAttempts' = failedAttempts,
    lockedUntil' = lockedUntil,
    activeSessions' = activeSessions,
  }

  action completePasswordReset(user: str, now: int) = all {
    USERS.contains(user),
    resetTokenStatus.get(user) == Pending,
    resetTokenExpiresAt.get(user) > now,
    resetTokenStatus' = resetTokenStatus.set(user, Used),
    failedAttempts' = failedAttempts.set(user, 0),
    status' = if (status.get(user) == Locked) status.set(user, Active) else status,
    lockedUntil' = if (status.get(user) == Locked) lockedUntil.set(user, 0) else lockedUntil,
    activeSessions' = activeSessions,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action resetTokenExpires(user: str, now: int) = all {
    USERS.contains(user),
    resetTokenStatus.get(user) == Pending,
    resetTokenExpiresAt.get(user) <= now,
    resetTokenStatus' = resetTokenStatus.set(user, Expired),
    status' = status,
    failedAttempts' = failedAttempts,
    lockedUntil' = lockedUntil,
    activeSessions' = activeSessions,
    resetTokenExpiresAt' = resetTokenExpiresAt,
  }

  action step = {
    nondet user = oneOf(USERS)
    nondet now = oneOf(NOW_VALUES)
    any {
      loginSuccess(user, now),
      loginFailure(user, now),
      lockoutExpires(user, now),
      logout(user),
      requestPasswordReset(user, now),
      completePasswordReset(user, now),
      resetTokenExpires(user, now),
    }
  }

  val attemptsNonNegative = USERS.forall(u => failedAttempts.get(u) >= 0)
  val sessionsNonNegative = USERS.forall(u => activeSessions.get(u) >= 0)
}
```

---

## Pattern 2: Role-Based Access Control (RBAC)

**Demonstrates:** hierarchical roles, permission inheritance, workspace membership operations

```quint
module RBAC {
  const USERS: Set[str]
  const WORKSPACES: Set[str]

  type Role = | Viewer | Editor | Admin

  var membershipRole: (str, str) -> Role
  var documentsPerWorkspace: str -> int

  pure def rolePermissions(role: Role): Set[str] =
    match role {
      | Viewer => Set("documents.read")
      | Editor => Set("documents.read", "documents.write")
      | Admin => Set("documents.read", "documents.write", "workspace.admin", "members.manage")
    }

  pure def canRead(user: str, workspace: str): bool =
    rolePermissions(membershipRole.get((workspace, user))).contains("documents.read")

  pure def canWrite(user: str, workspace: str): bool =
    rolePermissions(membershipRole.get((workspace, user))).contains("documents.write")

  pure def canAdmin(user: str, workspace: str): bool =
    rolePermissions(membershipRole.get((workspace, user))).contains("workspace.admin")

  action init = all {
    membershipRole' = WORKSPACES.cross(USERS).mapBy(_ => Viewer),
    documentsPerWorkspace' = WORKSPACES.mapBy(_ => 0),
  }

  action addMember(actor: str, workspace: str, newUser: str, role: Role) = all {
    canAdmin(actor, workspace),
    membershipRole' = membershipRole.set((workspace, newUser), role),
    documentsPerWorkspace' = documentsPerWorkspace,
  }

  action changeMemberRole(actor: str, workspace: str, user: str, role: Role) = all {
    canAdmin(actor, workspace),
    membershipRole' = membershipRole.set((workspace, user), role),
    documentsPerWorkspace' = documentsPerWorkspace,
  }

  action removeMember(actor: str, workspace: str, user: str) = all {
    canAdmin(actor, workspace),
    membershipRole' = membershipRole.set((workspace, user), Viewer),
    documentsPerWorkspace' = documentsPerWorkspace,
  }

  action createDocument(user: str, workspace: str) = all {
    canWrite(user, workspace),
    documentsPerWorkspace' = documentsPerWorkspace.setBy(workspace, n => n + 1),
    membershipRole' = membershipRole,
  }

  action step = {
    nondet actor = oneOf(USERS)
    nondet user = oneOf(USERS)
    nondet workspace = oneOf(WORKSPACES)
    nondet role = oneOf(Set(Viewer, Editor, Admin))
    any {
      addMember(actor, workspace, user, role),
      changeMemberRole(actor, workspace, user, role),
      removeMember(actor, workspace, user),
      createDocument(actor, workspace),
    }
  }

  val docCountNonNegative = WORKSPACES.forall(w => documentsPerWorkspace.get(w) >= 0)
}
```

---

## Pattern 3: Invitation to Resource

**Demonstrates:** invitation lifecycle, permission levels, accept/decline/revoke/expire

```quint
module ResourceInvitation {
  const RESOURCES: Set[str]
  const USERS: Set[str]
  const INVITATIONS: Set[int]
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

  var invites: int -> Invite
  var sharePermission: (str, str) -> SharePermission
  var shareActive: (str, str) -> bool

  action init = all {
    invites' = INVITATIONS.mapBy(_ => {
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

  action inviteToResource(invId: int, inviter: str, resource: str, email: str, permission: SharePermission, now: int) = all {
    INVITATIONS.contains(invId),
    invites.get(invId).status != Pending,
    invites' = invites.set(invId, {
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

  action acceptInvitationExistingUser(invId: int, user: str) = all {
    invites.get(invId).status == Pending,
    invites' = invites.setBy(invId, i => { resource: i.resource, email: i.email, permission: i.permission, invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Accepted }),
    sharePermission' = sharePermission.set((invites.get(invId).resource, user), invites.get(invId).permission),
    shareActive' = shareActive.set((invites.get(invId).resource, user), true),
  }

  action declineInvitation(invId: int) = all {
    invites.get(invId).status == Pending,
    invites' = invites.setBy(invId, i => { resource: i.resource, email: i.email, permission: i.permission, invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Declined }),
    sharePermission' = sharePermission,
    shareActive' = shareActive,
  }

  action invitationExpires(invId: int, now: int) = all {
    invites.get(invId).status == Pending,
    invites.get(invId).expiresAt <= now,
    invites' = invites.setBy(invId, i => { resource: i.resource, email: i.email, permission: i.permission, invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Expired }),
    sharePermission' = sharePermission,
    shareActive' = shareActive,
  }

  action revokeInvitation(invId: int) = all {
    invites.get(invId).status == Pending,
    invites' = invites.setBy(invId, i => { resource: i.resource, email: i.email, permission: i.permission, invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Revoked }),
    sharePermission' = sharePermission,
    shareActive' = shareActive,
  }

  action changeSharePermission(resource: str, user: str, permission: SharePermission) = all {
    shareActive.get((resource, user)),
    sharePermission' = sharePermission.set((resource, user), permission),
    invites' = invites,
    shareActive' = shareActive,
  }

  action revokeShare(resource: str, user: str) = all {
    shareActive.get((resource, user)),
    shareActive' = shareActive.set((resource, user), false),
    invites' = invites,
    sharePermission' = sharePermission,
  }

  action step = {
    nondet invId = oneOf(INVITATIONS)
    nondet user = oneOf(USERS)
    nondet resource = oneOf(RESOURCES)
    nondet now = oneOf(NOW_VALUES)
    nondet permission = oneOf(Set(View, Edit, Admin))
    any {
      inviteToResource(invId, user, resource, user, permission, now),
      acceptInvitationExistingUser(invId, user),
      declineInvitation(invId),
      invitationExpires(invId, now),
      revokeInvitation(invId),
      changeSharePermission(resource, user, permission),
      revokeShare(resource, user),
    }
  }
}
```

---

## Pattern 4: Soft Delete & Restore

**Demonstrates:** soft delete, restore permissions, retention-based hard delete

```quint
module SoftDelete {
  const DOCUMENTS: Set[int]
  const USERS: Set[str]
  const NOW_VALUES: Set[int]
  const RETENTION_PERIOD: int

  type DocStatus = | Active | Deleted | Purged

  var status: int -> DocStatus
  var deletedAt: int -> int
  var deletedBy: int -> str

  action init = all {
    status' = DOCUMENTS.mapBy(_ => Active),
    deletedAt' = DOCUMENTS.mapBy(_ => 0),
    deletedBy' = DOCUMENTS.mapBy(_ => ""),
  }

  action deleteDocument(actor: str, doc: int, now: int) = all {
    USERS.contains(actor),
    status.get(doc) == Active,
    status' = status.set(doc, Deleted),
    deletedAt' = deletedAt.set(doc, now),
    deletedBy' = deletedBy.set(doc, actor),
  }

  action restoreDocument(actor: str, doc: int, now: int) = all {
    status.get(doc) == Deleted,
    deletedAt.get(doc) + RETENTION_PERIOD > now,
    status' = status.set(doc, Active),
    deletedAt' = deletedAt.set(doc, 0),
    deletedBy' = deletedBy.set(doc, ""),
  }

  action permanentlyDelete(doc: int) = all {
    status.get(doc) == Deleted,
    status' = status.set(doc, Purged),
    deletedAt' = deletedAt,
    deletedBy' = deletedBy,
  }

  action retentionExpires(doc: int, now: int) = all {
    status.get(doc) == Deleted,
    deletedAt.get(doc) + RETENTION_PERIOD <= now,
    status' = status.set(doc, Purged),
    deletedAt' = deletedAt,
    deletedBy' = deletedBy,
  }

  action step = {
    nondet actor = oneOf(USERS)
    nondet doc = oneOf(DOCUMENTS)
    nondet now = oneOf(NOW_VALUES)
    any {
      deleteDocument(actor, doc, now),
      restoreDocument(actor, doc, now),
      permanentlyDelete(doc),
      retentionExpires(doc, now),
    }
  }
}
```

---

## Pattern 5: Notification Preferences & Digests

**Demonstrates:** notification variants, per-type email preference, immediate vs digest delivery

```quint
module Notifications {
  const USERS: Set[str]
  const NOTIFICATIONS: Set[int]
  const NOW_VALUES: Set[int]

  type EmailPreference = | Immediately | DailyDigest | Never
  type NotificationStatus = | Unread | Read | Archived
  type EmailStatus = | Pending | Sent | Skipped | Digested

  type NotificationKind =
    | Mention(commentId: int, mentionedBy: str)
    | Reply(replyId: int, originalCommentId: int, repliedBy: str)
    | Share(resource: str, sharedBy: str)
    | Assignment(taskId: int, assignedBy: str)
    | System(title: str)

  type Notification = {
    user: str,
    kind: NotificationKind,
    status: NotificationStatus,
    emailStatus: EmailStatus,
    createdAt: int,
  }

  var notification: int -> Notification
  var used: Set[int]

  var prefMention: str -> EmailPreference
  var prefReply: str -> EmailPreference
  var prefShare: str -> EmailPreference
  var prefAssignment: str -> EmailPreference
  var digestEnabled: str -> bool
  var nextDigestAt: str -> int

  action init = all {
    notification' = NOTIFICATIONS.mapBy(_ => {
      user: "",
      kind: System(""),
      status: Archived,
      emailStatus: Skipped,
      createdAt: 0,
    }),
    used' = Set(),
    prefMention' = USERS.mapBy(_ => Immediately),
    prefReply' = USERS.mapBy(_ => Immediately),
    prefShare' = USERS.mapBy(_ => Immediately),
    prefAssignment' = USERS.mapBy(_ => Immediately),
    digestEnabled' = USERS.mapBy(_ => true),
    nextDigestAt' = USERS.mapBy(_ => 24),
  }

  pure def preferenceFor(user: str, kind: NotificationKind): EmailPreference =
    match kind {
      | Mention(_, _) => prefMention.get(user)
      | Reply(_, _, _) => prefReply.get(user)
      | Share(_, _) => prefShare.get(user)
      | Assignment(_, _) => prefAssignment.get(user)
      | System(_) => Immediately
    }

  action createNotification(id: int, user: str, kind: NotificationKind, now: int) = all {
    NOTIFICATIONS.contains(id),
    not(used.contains(id)),
    notification' = notification.set(id, {
      user: user,
      kind: kind,
      status: Unread,
      emailStatus: if (preferenceFor(user, kind) == Never) Skipped else Pending,
      createdAt: now,
    }),
    used' = used.union(Set(id)),
    prefMention' = prefMention,
    prefReply' = prefReply,
    prefShare' = prefShare,
    prefAssignment' = prefAssignment,
    digestEnabled' = digestEnabled,
    nextDigestAt' = nextDigestAt,
  }

  action sendImmediateEmail(id: int) = all {
    used.contains(id),
    notification.get(id).emailStatus == Pending,
    preferenceFor(notification.get(id).user, notification.get(id).kind) == Immediately,
    notification' = notification.setBy(id, n => {
      user: n.user,
      kind: n.kind,
      status: n.status,
      emailStatus: Sent,
      createdAt: n.createdAt,
    }),
    used' = used,
    prefMention' = prefMention,
    prefReply' = prefReply,
    prefShare' = prefShare,
    prefAssignment' = prefAssignment,
    digestEnabled' = digestEnabled,
    nextDigestAt' = nextDigestAt,
  }

  action createDailyDigest(user: str, now: int) = all {
    digestEnabled.get(user),
    nextDigestAt.get(user) <= now,
    notification' = notification.setBy(id, n =>
      if (used.contains(id) and n.user == user and n.emailStatus == Pending)
      { user: n.user, kind: n.kind, status: n.status, emailStatus: Digested, createdAt: n.createdAt }
      else n),
    nextDigestAt' = nextDigestAt.set(user, now + 24),
    used' = used,
    prefMention' = prefMention,
    prefReply' = prefReply,
    prefShare' = prefShare,
    prefAssignment' = prefAssignment,
    digestEnabled' = digestEnabled,
  }

  action markRead(id: int) = all {
    used.contains(id),
    notification.get(id).status == Unread,
    notification' = notification.setBy(id, n => {
      user: n.user,
      kind: n.kind,
      status: Read,
      emailStatus: n.emailStatus,
      createdAt: n.createdAt,
    }),
    used' = used,
    prefMention' = prefMention,
    prefReply' = prefReply,
    prefShare' = prefShare,
    prefAssignment' = prefAssignment,
    digestEnabled' = digestEnabled,
    nextDigestAt' = nextDigestAt,
  }

  action step = {
    nondet id = oneOf(NOTIFICATIONS)
    nondet user = oneOf(USERS)
    nondet now = oneOf(NOW_VALUES)
    any {
      createNotification(id, user, Mention(1, user), now),
      createNotification(id, user, Reply(2, 1, user), now),
      createNotification(id, user, Share("doc", user), now),
      createNotification(id, user, Assignment(7, user), now),
      sendImmediateEmail(id),
      createDailyDigest(user, now),
      markRead(id),
    }
  }
}
```

---

## Pattern 6: Usage Limits & Quotas

**Demonstrates:** plan tiers, quota checks, rate limits, resets, upgrades/downgrades

```quint
module UsageLimits {
  const WORKSPACES: Set[str]
  const NOW_VALUES: Set[int]

  type Plan = | Free | Pro | Enterprise

  var plan: str -> Plan
  var documents: str -> int
  var members: str -> int
  var storageBytes: str -> int
  var apiRequestsToday: str -> int
  var nextResetAt: str -> int

  pure def maxDocuments(p: Plan): int =
    match p { | Free => 10 | Pro => 1000 | Enterprise => 1_000_000 }

  pure def maxMembers(p: Plan): int =
    match p { | Free => 3 | Pro => 20 | Enterprise => 1_000_000 }

  pure def maxApiRequests(p: Plan): int =
    match p { | Free => 100 | Pro => 10_000 | Enterprise => 1_000_000 }

  action init = all {
    plan' = WORKSPACES.mapBy(_ => Free),
    documents' = WORKSPACES.mapBy(_ => 0),
    members' = WORKSPACES.mapBy(_ => 1),
    storageBytes' = WORKSPACES.mapBy(_ => 0),
    apiRequestsToday' = WORKSPACES.mapBy(_ => 0),
    nextResetAt' = WORKSPACES.mapBy(_ => 24),
  }

  action createDocument(workspace: str) = all {
    documents.get(workspace) < maxDocuments(plan.get(workspace)),
    documents' = documents.setBy(workspace, n => n + 1),
    members' = members,
    storageBytes' = storageBytes,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
    plan' = plan,
  }

  action addMember(workspace: str) = all {
    members.get(workspace) < maxMembers(plan.get(workspace)),
    members' = members.setBy(workspace, n => n + 1),
    documents' = documents,
    storageBytes' = storageBytes,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
    plan' = plan,
  }

  action recordApiRequest(workspace: str) = all {
    apiRequestsToday.get(workspace) < maxApiRequests(plan.get(workspace)),
    apiRequestsToday' = apiRequestsToday.setBy(workspace, n => n + 1),
    documents' = documents,
    members' = members,
    storageBytes' = storageBytes,
    nextResetAt' = nextResetAt,
    plan' = plan,
  }

  action resetDailyApiUsage(workspace: str, now: int) = all {
    nextResetAt.get(workspace) <= now,
    apiRequestsToday' = apiRequestsToday.set(workspace, 0),
    nextResetAt' = nextResetAt.set(workspace, now + 24),
    documents' = documents,
    members' = members,
    storageBytes' = storageBytes,
    plan' = plan,
  }

  action upgradePlan(workspace: str, newPlan: Plan) = all {
    plan.get(workspace) != newPlan,
    plan' = plan.set(workspace, newPlan),
    documents' = documents,
    members' = members,
    storageBytes' = storageBytes,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
  }

  action downgradePlan(workspace: str, newPlan: Plan) = all {
    documents.get(workspace) <= maxDocuments(newPlan),
    members.get(workspace) <= maxMembers(newPlan),
    apiRequestsToday.get(workspace) <= maxApiRequests(newPlan),
    plan' = plan.set(workspace, newPlan),
    documents' = documents,
    members' = members,
    storageBytes' = storageBytes,
    apiRequestsToday' = apiRequestsToday,
    nextResetAt' = nextResetAt,
  }

  action step = {
    nondet workspace = oneOf(WORKSPACES)
    nondet now = oneOf(NOW_VALUES)
    nondet p = oneOf(Set(Free, Pro, Enterprise))
    any {
      createDocument(workspace),
      addMember(workspace),
      recordApiRequest(workspace),
      resetDailyApiUsage(workspace, now),
      upgradePlan(workspace, p),
      downgradePlan(workspace, p),
    }
  }
}
```

---

## Pattern 7: Comments with Mentions

**Demonstrates:** replies, mentions, mention/reply notification triggers, reactions

```quint
module Comments {
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

  action createComment(c: int, u: str, mentionedUsers: Set[str]) = all {
    COMMENTS.contains(c),
    USERS.contains(u),
    author' = author.set(c, u),
    replyTo' = replyTo.set(c, 0),
    mentionsByComment' = mentionsByComment.set(c, mentionedUsers),
    status' = status,
    reactions' = reactions,
    mentionEvents' = mentionEvents,
    replyEvents' = replyEvents,
  }

  action createReply(c: int, parent: int, u: str, mentionedUsers: Set[str]) = all {
    COMMENTS.contains(c),
    COMMENTS.contains(parent),
    status.get(parent) == Active,
    USERS.contains(u),
    author' = author.set(c, u),
    replyTo' = replyTo.set(c, parent),
    mentionsByComment' = mentionsByComment.set(c, mentionedUsers),
    replyEvents' = if (author.get(parent) != u and not(mentionedUsers.contains(author.get(parent))))
      replyEvents.union(Set((author.get(parent), c, parent)))
      else replyEvents,
    status' = status,
    reactions' = reactions,
    mentionEvents' = mentionEvents,
  }

  action notifyMentionedUser(c: int, mentioned: str) = all {
    mentionsByComment.get(c).contains(mentioned),
    mentioned != author.get(c),
    mentionEvents' = mentionEvents.union(Set((mentioned, c, author.get(c)))),
    status' = status,
    author' = author,
    replyTo' = replyTo,
    mentionsByComment' = mentionsByComment,
    reactions' = reactions,
    replyEvents' = replyEvents,
  }

  action editComment(c: int, newMentionedUsers: Set[str]) = all {
    status.get(c) == Active,
    mentionsByComment' = mentionsByComment.set(c, newMentionedUsers),
    status' = status,
    author' = author,
    replyTo' = replyTo,
    reactions' = reactions,
    mentionEvents' = mentionEvents,
    replyEvents' = replyEvents,
  }

  action deleteComment(c: int) = all {
    status.get(c) == Active,
    status' = status.set(c, Deleted),
    author' = author,
    replyTo' = replyTo,
    mentionsByComment' = mentionsByComment,
    reactions' = reactions,
    mentionEvents' = mentionEvents,
    replyEvents' = replyEvents,
  }

  action toggleReaction(c: int, u: str, emoji: str) = all {
    status.get(c) == Active,
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

---

## Pattern 8: Integrating Library Specs

**Demonstrates:** external module composition for OAuth and billing workflows

### Example A: OAuth-backed app authentication

```quint
module AppAuth {
  import OAuth.* from "oauth2"

  const USERS: Set[str]

  type UserStatus = | Active | Suspended | Deactivated

  var status: str -> UserStatus
  var lastLoginAt: str -> int
  var linkedProviders: str -> Set[str]

  action init = all {
    status' = USERS.mapBy(_ => Active),
    lastLoginAt' = USERS.mapBy(_ => 0),
    linkedProviders' = USERS.mapBy(_ => Set()),
  }

  action createUserOnFirstLogin(user: str, provider: str, now: int) = all {
    status.get(user) == Active,
    linkedProviders' = linkedProviders.setBy(user, ps => ps.union(Set(provider))),
    lastLoginAt' = lastLoginAt.set(user, now),
    status' = status,
  }

  action updateUserOnLogin(user: str, now: int) = all {
    status.get(user) == Active,
    lastLoginAt' = lastLoginAt.set(user, now),
    status' = status,
    linkedProviders' = linkedProviders,
  }

  action blockSuspendedUserLogin(user: str) = all {
    status.get(user) == Suspended,
    status' = status,
    lastLoginAt' = lastLoginAt,
    linkedProviders' = linkedProviders,
  }

  action linkAdditionalProvider(user: str, provider: str) = all {
    status.get(user) == Active,
    not(linkedProviders.get(user).contains(provider)),
    linkedProviders' = linkedProviders.setBy(user, ps => ps.union(Set(provider))),
    status' = status,
    lastLoginAt' = lastLoginAt,
  }

  action unlinkProvider(user: str, provider: str) = all {
    status.get(user) == Active,
    linkedProviders.get(user).size() > 1,
    linkedProviders' = linkedProviders.setBy(user, ps => ps.exclude(provider)),
    status' = status,
    lastLoginAt' = lastLoginAt,
  }
}
```

### Example B: Billing with imported payment module

```quint
module Billing {
  import StripeBilling.* from "stripe-billing"

  const ORGS: Set[str]

  type SubscriptionStatus = | Trialing | Active | PastDue | Cancelled | Expired
  type Plan = | Free | Pro | Enterprise

  var subStatus: str -> SubscriptionStatus
  var subPlan: str -> Plan
  var trialReminderSent: str -> bool

  action init = all {
    subStatus' = ORGS.mapBy(_ => Trialing),
    subPlan' = ORGS.mapBy(_ => Free),
    trialReminderSent' = ORGS.mapBy(_ => false),
  }

  action activateOnPaymentSuccess(org: str) = all {
    Set(Trialing, PastDue).contains(subStatus.get(org)),
    subStatus' = subStatus.set(org, Active),
    subPlan' = subPlan,
    trialReminderSent' = trialReminderSent,
  }

  action handlePaymentFailure(org: str) = all {
    subStatus' = subStatus.set(org, PastDue),
    subPlan' = subPlan,
    trialReminderSent' = trialReminderSent,
  }

  action trialEndingReminder(org: str) = all {
    subStatus.get(org) == Trialing,
    not(trialReminderSent.get(org)),
    trialReminderSent' = trialReminderSent.set(org, true),
    subStatus' = subStatus,
    subPlan' = subPlan,
  }

  action handleSubscriptionCancelled(org: str) = all {
    subStatus' = subStatus.set(org, Cancelled),
    subPlan' = subPlan,
    trialReminderSent' = trialReminderSent,
  }

  action startSubscription(org: str, p: Plan) = all {
    Set(Cancelled, Expired).contains(subStatus.get(org)),
    subStatus' = subStatus.set(org, Trialing),
    subPlan' = subPlan.set(org, p),
    trialReminderSent' = trialReminderSent.set(org, false),
  }

  action changePlan(org: str, p: Plan) = all {
    subStatus.get(org) == Active,
    p != subPlan.get(org),
    subPlan' = subPlan.set(org, p),
    subStatus' = subStatus,
    trialReminderSent' = trialReminderSent,
  }

  action cancelSubscription(org: str) = all {
    Set(Active, Trialing).contains(subStatus.get(org)),
    subStatus' = subStatus.set(org, Cancelled),
    subPlan' = subPlan,
    trialReminderSent' = trialReminderSent,
  }
}
```

## Using these patterns

1. Start with the nearest pattern and rename entities to your domain.
2. Keep finite sets and bounded counters while shaping behaviour.
3. Add invariants first, then temporal properties.
4. Compose patterns by importing modules and sharing keys (user, workspace, resource IDs).

## Additional parity snippets

These snippets keep extra functional examples that were present in the earlier catalog.

### Password auth: locked-attempt and session-expiry paths

```quint
action loginAttemptWhileLocked(user: str, now: int) = all {
  status.get(user) == Locked,
  lockedUntil.get(user) > now,
  // user informed; state unchanged in core auth maps
  status' = status,
  failedAttempts' = failedAttempts,
  lockedUntil' = lockedUntil,
  activeSessions' = activeSessions,
  resetTokenStatus' = resetTokenStatus,
  resetTokenExpiresAt' = resetTokenExpiresAt,
}

action sessionExpires(user: str) = all {
  activeSessions.get(user) > 0,
  activeSessions' = activeSessions.setBy(user, n => n - 1),
  status' = status,
  failedAttempts' = failedAttempts,
  lockedUntil' = lockedUntil,
  resetTokenStatus' = resetTokenStatus,
  resetTokenExpiresAt' = resetTokenExpiresAt,
}
```

### RBAC: permission maintenance and self-leave flow

```quint
action leaveWorkspace(user: str, workspace: str) = all {
  membershipRole.get((workspace, user)) != Viewer,
  membershipRole' = membershipRole.set((workspace, user), Viewer),
  documentsPerWorkspace' = documentsPerWorkspace,
}

action grantEditorFromViewer(actor: str, workspace: str, user: str) = all {
  canAdmin(actor, workspace),
  membershipRole.get((workspace, user)) == Viewer,
  membershipRole' = membershipRole.set((workspace, user), Editor),
  documentsPerWorkspace' = documentsPerWorkspace,
}
```

### Resource invitation: new-user acceptance and invitation revocation

```quint
var users: Set[str]

action acceptInvitationNewUser(invId: int, newUser: str) = all {
  invites.get(invId).status == Pending,
  not(users.contains(newUser)),
  users' = users.union(Set(newUser)),
  invites' = invites.setBy(invId, i => {
    resource: i.resource, email: i.email, permission: i.permission,
    invitedBy: i.invitedBy, expiresAt: i.expiresAt, status: Accepted
  }),
  shareActive' = shareActive.set((invites.get(invId).resource, newUser), true),
  sharePermission' = sharePermission.set((invites.get(invId).resource, newUser), invites.get(invId).permission),
}
```

### Soft delete: empty trash and restore-all operations

```quint
action emptyTrash(now: int) = all {
  status' = DOCUMENTS.mapBy(doc =>
    if (status.get(doc) == Deleted and deletedAt.get(doc) + RETENTION_PERIOD <= now)
      Purged
      else status.get(doc)),
  deletedAt' = deletedAt,
  deletedBy' = deletedBy,
}

action restoreAll(now: int) = all {
  status' = DOCUMENTS.mapBy(doc =>
    if (status.get(doc) == Deleted and deletedAt.get(doc) + RETENTION_PERIOD > now)
      Active
      else status.get(doc)),
  deletedAt' = DOCUMENTS.mapBy(doc =>
    if (status.get(doc) == Deleted and deletedAt.get(doc) + RETENTION_PERIOD > now)
      0
      else deletedAt.get(doc)),
  deletedBy' = deletedBy,
}
```

### Notifications: archive and mark-all-read

```quint
action archiveNotification(id: int) = all {
  used.contains(id),
  notification' = notification.setBy(id, n => {
    user: n.user, kind: n.kind, status: Archived, emailStatus: n.emailStatus, createdAt: n.createdAt
  }),
  used' = used,
}

action markAllRead(user: str) = all {
  notification' = NOTIFICATIONS.mapBy(id =>
    if (used.contains(id) and notification.get(id).user == user and notification.get(id).status == Unread)
      {
        user: notification.get(id).user,
        kind: notification.get(id).kind,
        status: Read,
        emailStatus: notification.get(id).emailStatus,
        createdAt: notification.get(id).createdAt
      }
      else notification.get(id)),
  used' = used,
}
```

### Usage limits: downgrade blocking semantics

```quint
action downgradeBlocked(workspace: str, newPlan: Plan) = all {
  documents.get(workspace) > maxDocuments(newPlan)
    or members.get(workspace) > maxMembers(newPlan)
    or apiRequestsToday.get(workspace) > maxApiRequests(newPlan),
  // explicitly blocked: state unchanged
  plan' = plan,
  documents' = documents,
  members' = members,
  storageBytes' = storageBytes,
  apiRequestsToday' = apiRequestsToday,
  nextResetAt' = nextResetAt,
}
```

### Comments: explicit remove-reaction variant

```quint
action removeReaction(c: int, u: str, emoji: str) = all {
  reactions.contains((c, u, emoji)),
  reactions' = reactions.exclude((c, u, emoji)),
  status' = status,
  author' = author,
  replyTo' = replyTo,
  mentionsByComment' = mentionsByComment,
  mentionEvents' = mentionEvents,
  replyEvents' = replyEvents,
}
```

### Integration modules: suspend-login policy remains explicit

```quint
action blockSuspendedUserLogin(user: str) = all {
  status.get(user) == Suspended,
  // state unchanged: login is denied by guard
  status' = status,
  lastLoginAt' = lastLoginAt,
  linkedProviders' = linkedProviders,
}
```
