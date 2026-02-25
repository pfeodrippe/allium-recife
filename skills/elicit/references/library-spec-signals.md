# Recognizing library module opportunities

During elicitation, identify behavior that should live in a reusable P module rather than inline application logic.

## Signals that something might be a reusable module

### External system integration

Examples:

- OAuth / OIDC sign-in flows
- payment provider workflows
- webhook processing with retries/signature checks
- file storage and scanning lifecycle

### Generic protocol patterns

Examples:

- request/ack/commit
- lease acquire/renew/release
- heartbeat/suspect/fail

### Implementation-agnostic requirement language

If stakeholders describe category behavior independent of vendor, the boundary is usually reusable.

## Questions to ask

1. Would another system use this same protocol?
2. Are customization points limited and explicit?
3. Is provider choice expected to change without product redesign?
4. Is this behavior primarily integration mechanics rather than domain policy?

## How to handle the decision

### Option 1: Use existing reusable module

Prefer reuse when protocol behavior is standard and already modeled.

### Option 2: Create new reusable module

Extract if behavior is generic and likely to recur across projects.

### Option 3: Keep inline

Keep inline when behavior is domain-specific and unlikely to be reused.

## Common reusable module candidates

- Authentication/SSO
- Payment lifecycle
- Notification delivery channel management
- Queue retry/dead-letter logic
- Webhook verification and retry protocol

## The boundary question

Reusable module should own:

- protocol-level event flow
- retry/timeout/ordering semantics
- protocol invariants and liveness obligations

Application module should own:

- domain-specific decisions after protocol events
- domain entities and policy checks
- user-visible product behavior

### Boundary example

```p
module AuthProtocol = { OAuthBoundary, SessionLifecycle };
module AppDomain = { UserProvisioning, RoleAssignment };

module FullSystem = (union AuthProtocol, AppDomain);
module CheckedSystem = assert AuthSafety, AppSafety in FullSystem;
```

## Red flags you missed a module opportunity

- same protocol logic copied across features
- repeated monitor definitions for the same integration mechanics
- domain machines bloated with third-party lifecycle details
