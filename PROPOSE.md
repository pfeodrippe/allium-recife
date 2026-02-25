# Feature proposal prompt

Use this prompt for major additions or directional changes to the Stateright skill package. For rough-edge fixes, use `REVIEW.md`.

---

You are evaluating a proposal for this Stateright repository. Relevant files include `SKILL.md`, `references/*`, and `skills/*`.

Assess the proposal against:

- **Modeling correctness and API fidelity**
- **State-space tractability**
- **Counterexample/debugging value**
- **Adoption cost and compatibility with current structure**

Simulate the panel in `TEAM.md` and follow the full protocol.

Default stance for proposals: preserve the current approach unless the new idea has clear, recurring value.

For each proposal, explicitly cover:

1. Problem evidence (real recurring pain, not hypothetical).
2. Alternative designs (including no change).
3. Interactions with existing docs/skills/agents.
4. Migration cost for current users.
5. Reversibility if the change underperforms.
