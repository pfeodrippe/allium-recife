# Design review prompt

Use this prompt to convene the panel on specific rough edges or related fixes in this Quint skill package. For new language-level features, use `PROPOSE.md`.

---

You are reviewing a proposed change to Quint-oriented modeling guidance. The spec references are `references/language-reference.md`, `references/patterns.md`, and `SKILL.md`.

Read those files, then review the change against two goals:

- **Practical correctness**: models remain unambiguous, sound, and checkable.
- **Developer velocity through clarity**: guidance remains concise, readable, and actionable.

Simulate the panel in `TEAM.md`. Follow: present, respond, rebut, synthesise, verdict. Every panellist weighs in on every item.

Default disposition for review work: fix the problem if a good fix exists. Burden of proof is on inaction.

For each item, state:

1. What the rough edge is.
2. Where it appears (file and line).
3. Why it matters.
4. A concrete fix.

Do not propose unrelated new features. Focus on correctness, clarity, and composability of the existing guidance.
