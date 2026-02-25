# Feature proposal prompt

Use this prompt to convene the review panel on a proposed Quint-language feature, extension, or ambitious change. For fixing rough edges in existing guidance, use `REVIEW.md`.

---

You are evaluating a proposed feature or extension to Quint-oriented modeling guidance in this repository. The reference lives in `references/language-reference.md`, with reusable patterns in `references/patterns.md` and authoring guidance in `SKILL.md`.

Read the full language reference, patterns file, and `SKILL.md`. Evaluate the proposal against two goals:

- **Practical correctness**: models are unambiguous, checkable, and hard to misread.
- **Developer velocity through clarity**: models are fast to author, easy to review, and cheap to change.

Simulate the design review panel described in `TEAM.md`. Follow the debate protocol in that file: present, respond, rebut, synthesise, verdict. Every panellist must weigh in. Produce the report using the output format in `TEAM.md`.

Default disposition for proposals: leave the language guidance unchanged unless evidence is strong. Burden of proof is on the proposal.

For each proposal, address:

1. **Problem.** What recurring limitation does this solve? Is it concrete?
2. **Design space.** What alternatives exist, including doing nothing?
3. **Interactions.** How does it compose with current Quint patterns and tooling?
4. **Cost.** Learning cost, maintenance cost, and tooling impact.
5. **Reversibility.** How hard is rollback if the idea underperforms?
