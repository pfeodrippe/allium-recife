# Complete patterns

Reusable Stateright patterns for common product and distributed-system behavior.

Patterns intentionally omit infrastructure plumbing and focus on state, actions, and properties.

| Pattern | Key features demonstrated |
|---------|---------------------------|
| Password auth with reset | token lifecycle, expiry, replay safety |
| Role-based access control | permission derivation, authorization invariants |
| Invitation workflow | acceptance/decline/expiry transitions |
| Soft delete and restore | reversible lifecycle and visibility guarantees |
| Notification preferences and digests | user policy + scheduled batching |
| Usage limits and quotas | bounded resources + guard enforcement |
| Comments with mentions | parsing events + notification fan-out |
| Integrating reusable model components | composing shared protocol fragments |

---

## Pattern 1: Password auth with reset

**Demonstrates:** status machine, token expiry, replay prevention.

State sketch:

```rust
enum UserStatus { Active, Locked, Deactivated }
enum TokenStatus { Pending, Used, Expired }
```

Core actions:

- `RequestReset { user_id }`
- `CompleteReset { token_id }`
- `ExpireToken { token_id }`

Properties:

- `always`: pending token cannot be used after expiry.
- `always`: completing reset invalidates token.
- `sometimes`: successful reset is reachable.

---

## Pattern 2: Role-based access control (RBAC)

**Demonstrates:** role assignment and permission checks.

State sketch:

```rust
struct User { roles: BTreeSet<u8> }
struct Role { permissions: BTreeSet<u8> }
```

Core actions:

- `AssignRole { user_id, role_id }`
- `RevokeRole { user_id, role_id }`
- `AttemptOperation { user_id, operation_id }`

Properties:

- `always`: operation requires matching permission.
- `always`: revocation removes access unless another role still grants it.
- `sometimes`: authorized operation succeeds when permission exists.

---

## Pattern 3: Invitation workflow

**Demonstrates:** invitation lifecycle and timeout behavior.

State sketch:

```rust
enum InvitationStatus { Pending, Accepted, Declined, Expired }
```

Core actions:

- `SendInvitation`
- `AcceptInvitation`
- `DeclineInvitation`
- `ExpireInvitation`

Properties:

- `always`: accepted invitation cannot later be declined/expired.
- `always`: pending invitation eventually leaves pending under expiry action availability.
- `sometimes`: acceptance path exists before expiry.

---

## Pattern 4: Soft delete and restore

**Demonstrates:** reversible deletion while preserving invariants.

State sketch:

```rust
enum ItemStatus { Active, Deleted }
```

Core actions:

- `DeleteItem`
- `RestoreItem`
- `ReadVisibleItems`

Properties:

- `always`: deleted items are excluded from active views.
- `always`: restore only applies to deleted items.
- `sometimes`: item can be deleted then restored.

---

## Pattern 5: Notification preferences and digests

**Demonstrates:** policy-driven dispatch and batch schedules.

State sketch:

```rust
enum DeliveryMode { Immediate, Digest, Muted }
struct PendingEvent { user_id: u8, kind: u8 }
```

Core actions:

- `EmitEvent`
- `DispatchImmediate`
- `RunDigestBatch`

Properties:

- `always`: muted users receive no notification actions.
- `always`: digest mode defers until batch action.
- `sometimes`: digest delivery occurs for queued events.

---

## Pattern 6: Usage limits and quotas

**Demonstrates:** guard checks with resource counters.

State sketch:

```rust
struct Workspace { used_projects: u8, max_projects: u8 }
```

Core actions:

- `CreateProject`
- `DeleteProject`

Properties:

- `always`: `used_projects <= max_projects`.
- `always`: create is disabled when limit reached.
- `sometimes`: project creation is reachable below quota.

---

## Pattern 7: Comments with mentions

**Demonstrates:** parsing-derived effects and deduplicated fan-out.

State sketch:

```rust
struct Comment { author_id: u8, body: u8 }
struct MentionNotif { comment_id: u8, user_id: u8 }
```

Core actions:

- `AddComment`
- `ExtractMentions`
- `CreateMentionNotification`

Properties:

- `always`: author is not self-notified unless explicitly configured.
- `always`: one notification per `(comment, user)` pair.
- `sometimes`: mention notification is reachable when mention exists.

---

## Pattern 8: Integrating reusable model components

**Demonstrates:** composition of shared mechanics with app-specific policy.

Example components:

- retry envelope
- lease manager
- idempotency cache

Composition rule:

- shared component owns protocol mechanics and core safety properties
- app model owns domain policy and business constraints

Properties:

- `always`: component invariants remain true after composition.
- `always`: app-level invariants hold with component actions interleaved.

---

## Using these patterns

### Composition

Start with one isolated pattern and validate its properties. Then compose patterns incrementally and add one cross-pattern invariant per merge.

### Adaptation

Rename entities/actions to domain terms, but keep transition and property intent intact.

### Anti-patterns

- copying pattern code without adapting bounds
- importing only actions without invariant set
- composing multiple patterns before isolated validation
