# Design review panel

Every proposed change to this P skill pack is reviewed by a nine-member panel. Each panellist highlights a different risk so tradeoffs are explicit before guidance is adopted.

## The panellists

### The simplicity advocate

Pushes for the fewest constructs needed for clear P modeling guidance.

### The machine reasoning advocate

Prioritizes consistent syntax and structure that LLMs can follow reliably.

### The composability advocate

Ensures guidance scales through modules, substitution, and reusable patterns.

### The readability advocate

Protects clarity for engineers who did not author the model.

### The rigour advocate

Checks semantic correctness against real P execution and checker behavior.

### The domain modelling advocate

Keeps domain intent central and avoids protocol-only tunnel vision.

### The developer experience advocate

Optimizes for practical adoption and short correction loops.

### The creative advocate

Looks for higher-leverage reframings when incremental fixes are insufficient.

### The backward compatibility advocate

Evaluates migration cost and disruption for existing users of this skill pack.

## How the debate works

### Scope

Applies to focused reviews and broader proposals.

### Protocol

1. Present: state change, motivation, and expected impact.
2. Respond: each panellist provides concise feedback.
3. Rebut: one round on unresolved objections.
4. Synthesise: neutral summary of resolved and unresolved points.
5. Verdict: `adopt`, `refine`, `reject`, or `split`.

### Verdicts

- `adopt`: objections resolved or accepted as explicit tradeoffs.
- `refine`: direction is valid but wording/scope needs another pass.
- `reject`: cost/risk exceeds benefit.
- `split`: unresolved strategic disagreement, escalate.

### Consensus and authority

Consensus means objections were addressed or acknowledged as acceptable tradeoffs. It does not require full enthusiasm.

Splits are escalated with strongest arguments from both sides recorded.

### Output format

1. Summary
2. Items debated
3. Deferred items (`split` or unresolved)
