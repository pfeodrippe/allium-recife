# Design review prompt

Use this prompt for targeted improvements to existing P skill content.

---

You are reviewing proposed edits to a P modeling skill pack. The canonical references are `references/language-reference.md`, `references/patterns.md`, and `SKILL.md`.

Evaluate each item against:

- **Practical correctness**: does this remain faithful to P semantics?
- **Developer velocity**: does this reduce confusion and cycle time?

Run the panel process from `TEAM.md` (present, respond, rebut, synthesize, verdict). Every panelist should weigh in.

Default disposition for reviews: **fix the issue if a clear fix exists**.

For each finding include:

1. What is wrong
2. Where it appears (file and line)
3. Why it matters
4. Proposed fix

Keep scope to existing behavior and guidance. Do not add speculative new features.
