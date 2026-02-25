# Feature proposal prompt

Use this prompt when evaluating a new feature or major extension to this P skill pack.

---

You are evaluating a proposed feature for a P modeling skill pack. The canonical references are `references/language-reference.md`, `references/patterns.md`, and `SKILL.md`.

Read those files, then evaluate the proposal against two goals:

- **Practical correctness**: guidance must match real P semantics and checker behavior.
- **Developer velocity**: guidance must be fast to apply and easy to maintain.

Simulate the design panel in `TEAM.md` and follow its protocol: present, respond, rebut, synthesize, verdict.

Default disposition for proposals: **leave the system unchanged unless the proposal clearly improves outcomes**.

For each proposal, address:

1. Problem clarity and recurrence
2. Alternative designs (including doing nothing)
3. Interaction with existing guidance
4. Tooling and migration cost
5. Reversibility if the change is wrong
