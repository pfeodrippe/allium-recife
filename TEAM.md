# Design review panel

Major changes to this Stateright skill pack are debated by a nine-role panel before adoption. Each role emphasizes a different modeling risk so decisions are not optimized for a single viewpoint.

## The panellists

### The state-space pragmatist

Guards tractability. Asks whether the proposal keeps models checkable under practical bounds. Rejects elegant guidance that drives unavoidable explosion without proportional bug-finding value.

### The safety advocate

Prioritizes invariants and forbidden-state checks. Pushes for `Property::always` coverage before optimistic progress claims. Flags proposals that make safety checks harder to express or maintain.

### The progress advocate

Protects progress semantics. Asks whether reachability/eventuality intent is represented clearly and whether assumptions are explicit. Pushes back on vague liveness language.

### The API fidelity advocate

Checks alignment with real Stateright APIs and current idioms. Rejects invented helper functions or stale API assumptions that would mislead users.

### The composability advocate

Asks whether guidance composes across patterns and domains. Favors modular model structure that supports reuse of shared mechanics (retries, leases, dedup).

### The readability advocate

Optimizes for fast comprehension by engineers reading model code and traces. Rejects wording that is technically correct but operationally confusing.

### The developer experience advocate

Optimizes iteration loop quality: author model, run checker, inspect trace, patch. Cares about error diagnosability and how quickly users recover from mistakes.

### The creative advocate

Looks for better modeling abstractions when recurring pain appears. Encourages stronger pattern extraction and reusable model components when it improves signal.

### The backward compatibility advocate

Protects users of current repository structure and workflows (`SKILL`, `skills/elicit`, `skills/distill`, `references`, `.claude`). Requires clear migration notes when behavior changes.

## How the debate works

### Scope

The debate protocol applies to both:

- **reviews**: rough-edge fixes to current guidance
- **proposals**: new capabilities, patterns, or direction shifts

### Protocol

1. **Present.** State the problem and candidate change.
2. **Respond.** Every panellist gives a concise position.
3. **Rebut.** One focused rebuttal round across roles.
4. **Synthesise.** Summarize remaining disagreements neutrally.
5. **Verdict.** Choose outcome and required follow-up.

### Verdicts

- **Consensus: adopt.** Concerns are addressed or accepted as explicit tradeoffs.
- **Consensus: reject.** Proposal is weaker than status quo.
- **Refine.** Promising direction with fixable gaps; run one more cycle.
- **Split.** Irreconcilable tradeoff; document both sides.

### Consensus and authority

The panel does not vote by count. Consensus means unresolved objections are either addressed or explicitly accepted as tradeoffs. Splits are escalated to maintainers with both positions captured faithfully.

### Output format

For each debated item include:

1. problem statement and affected files
2. key tensions by panellist role
3. verdict and rationale
4. exact edits required for adopted items
5. deferred questions for refine/split outcomes
