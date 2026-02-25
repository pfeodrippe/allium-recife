# Recognizing reusable model-component opportunities

During elicitation, identify behavior that should be modeled as reusable Stateright components instead of inlined one-off logic.

## Signals

### 1. Repeated protocol mechanics

Examples:

- request/ack/retry loops
- leases and renewals
- quorum vote counting
- idempotent command dedup

If the same mechanics appear across domains, extract a reusable model pattern.

### 2. Integration-independent behavior

If behavior is not tied to one product domain (e.g., generic retry/backoff), model it as a reusable component.

### 3. Same invariants across teams

If multiple systems require the same invariants, create a shared model template with property suite.

## Questions to ask

1. Would another team model this the same way?
2. Are invariants generic rather than domain-specific?
3. Is this primarily protocol behavior, not business policy?

If yes to most, factor out.

## Handling options

### Option A: Reuse existing pattern

Adopt an existing pattern from `references/patterns.md` and tune bounds/config.

### Option B: Create a reusable module in project code

Extract state/action/property fragments into a local shared model crate/module.

### Option C: Keep inline

Only when behavior is truly product-specific and unlikely to recur.

## Boundary rule

Reusable component should own:

- protocol mechanics
- generic safety properties
- parameterized bounds

Application model should own:

- domain entities/policies
- product-specific constraints
- integration choices

## Red flags you missed reuse

- copying action enums across models
- same property predicates rewritten in multiple repos
- recurring counterexamples with identical root cause shape
