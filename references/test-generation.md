# Test generation

From a Recife model (`.clj`), generate:

**Process transition tests** (per `defproc`):
- Success case: guard conditions hold, verify expected state transition
- Failure cases: one test per guard clause, verify no transition (`nil` or unchanged state)
- Edge cases: boundary values for time, counts and quotas

**State transition tests** (per entity/status map):
- Valid transitions succeed via process steps
- Invalid transitions are rejected (guards fail)
- Terminal states have no outbound transitions

**Temporal tests** (per time-based process/property):
- Before deadline: no transition
- At deadline: transition fires and updates state
- After deadline: idempotent behavior (no duplicate effect)

**Communication/event tests** (per outbox queue):
- Verify event is emitted
- Verify recipient/target is correct
- Verify payload shape is complete

**Scenario tests** (per flow):
- Happy path through main flow
- Edge cases and error paths
- Concurrent scenarios: process interleavings still satisfy invariants

**Tagged-state (sum-type-by-convention) tests**:
- Tag discrimination (`:kind`/`:type`) routes logic to correct branch
- Exhaustiveness in `case`/`cond`
- Invalid tag values are blocked by invariants

**Boundary contract tests** (surface-equivalent state slices):
- Visibility tests for exposed fields
- Operation availability by role/scope
- Rejection tests when preconditions are missing
- Navigation/relationship consistency tests

**Cross-process interaction tests**:
- Re-trigger sibling processes while dependent state exists
- Verify guards prevent duplicate creation/conflicting state
- Verify command queues are consumed deterministically

**Concurrency note:** Recife process transitions are atomic per step. If two processes can apply on the same state, test both interleavings and verify invariants/properties still hold.
