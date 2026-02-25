# Modeling review prompt

Use this prompt to review fixes to rough edges in the Stateright skill package. For new feature ideas, use `PROPOSE.md`.

---

You are reviewing a change to this Stateright-focused repository. Core files are `SKILL.md`, `references/language-reference.md`, `references/patterns.md`, `skills/elicit/SKILL.md`, and `skills/distill/SKILL.md`.

Read the relevant files and evaluate the change against two goals:

- **Practical correctness:** guidance must produce valid, checkable models.
- **Velocity through clarity:** users should move quickly from intent/code to executable checks.

Simulate the panel in `TEAM.md` and follow its protocol (present, respond, rebut, synthesize, verdict).

Default stance for reviews: fix the issue if a concrete, low-risk fix exists.

For each item, include:

1. What is broken or confusing.
2. Where it appears (file and line).
3. Why it matters to model quality or debugging.
4. Candidate fix with concrete edits.
