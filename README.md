# P

*Velocity through executable clarity*

---

A skill pack for modeling, checking, and evolving distributed system behavior with [P](https://p-org.github.io/P/).

P is a state-machine language for event-driven systems. It lets you model components as communicating machines, define safety and liveness properties as monitors, and check behavior under many schedules with `p check`.

## Get started

**Claude Code** (plugin marketplace):

```text
/plugin marketplace add juxt/claude-plugins
/plugin install p
```

**Cursor, Windsurf, Copilot, Aider, Continue and other skills-compatible tools:**

```text
npx skills add juxt/p
```

Once installed, run `/p` for general support.

- `/p:elicit` builds a model from stakeholder conversations.
- `/p:distill` extracts a model from existing code.

## Install P CLI

This repository also assumes the upstream `p` compiler/checker CLI is available.

Requirements:

- .NET SDK 8.x
- Java runtime 11+

Install:

```bash
dotnet tool install --global P
```

Update (if already installed):

```bash
dotnet tool update --global P
```

Verify:

```bash
p --help
```

Official install/usage docs:

- https://p-org.github.io/P/getstarted/install/
- https://p-org.github.io/P/getstarted/usingP/

## Working with agents

A rule at [.claude/rules/p.md](.claude/rules/p.md) auto-loads for `.p` files and keeps syntax and checker semantics in view.

Two specialized agents support delegated workflows:

- **[tend](.claude/agents/tend.md)** grows and refactors P models (`.p`, `.pproj`) from requirement changes.
- **[weed](.claude/agents/weed.md)** compares P models and implementation code, reports drift, and can update either side.

Use `/p` for direct language help, `elicit` for requirement capture, `distill` for code-to-model extraction, `tend` for model evolution, and `weed` for drift detection.

## The problem with conversational context

- In long chats, assumptions drift and implicit constraints get blurred.
- Across sessions, critical behavioral intent is easy to lose.

P gives intent a durable executable form. You do not rely on memory or prompt archaeology to preserve correctness conditions.

## Why not just point the LLM at the code?

Code tells you what currently exists, including accidental behavior and historical workarounds.

P models force a separate, explicit statement of intended protocol behavior and system properties. When model and code disagree, that disagreement is useful signal.

## Why not capture requirements in markdown?

Markdown can describe behavior, but it does not execute, type-check, or explore interleavings.

P adds structure that tools can validate:

- typed events and payloads
- explicit state transitions
- executable safety and liveness monitors
- systematic schedule exploration

## Iterating on specifications

P works best as a companion artifact, not a one-off document.

- **Elicitation** captures intended behavior before or during implementation.
- **Distillation** captures actual behavior from code and runtime logic.

The two meet in review: if they differ, decide which side changes.

## On single sources of truth

Code and model are different truths about different concerns:

- code: implementation mechanics
- model: intended distributed behavior and properties

Maintaining both is similar to maintaining tests and types. It is deliberate redundancy that catches drift earlier.

## What P captures

P captures distributed behavior in terms of events, state machines, modules, tests, and monitors.

```p
type tTransfer = (id: int, from: int, to: int, amount: int, client: machine);
type tTransferResult = (id: int, ok: bool);

event eTransferReq: tTransfer;
event eTransferResp: tTransferResult;

event eLedgerApplied: (id: int, fromBalance: int, toBalance: int);

machine Ledger {
  var balance: map[int, int];

  start state Ready {
    on eTransferReq do (req: tTransfer) {
      if (balance[req.from] >= req.amount) {
        balance[req.from] = balance[req.from] - req.amount;
        balance[req.to] = balance[req.to] + req.amount;
        send req.client, eTransferResp, (id = req.id, ok = true);
        announce eLedgerApplied, (id = req.id, fromBalance = balance[req.from], toBalance = balance[req.to]);
      } else {
        send req.client, eTransferResp, (id = req.id, ok = false);
      }
    }
  }
}

spec NonNegativeBalances observes eLedgerApplied {
  start state S {
    on eLedgerApplied do (x: (id: int, fromBalance: int, toBalance: int)) {
      assert x.fromBalance >= 0 && x.toBalance >= 0,
        "negative balance observed";
    }
  }
}
```

### A language with a checker

P has a compiler and a checker.

- `p compile` builds a model executable from your `.p` files.
- `p check` explores schedules, checks assertions, deadlocks, unhandled events, and monitor properties.

This turns "does this seem right?" into a concrete verification workflow.

## What this looks like in practice

### P surfaces implications you missed

You ask for retries on timeout.

The model reveals retries can duplicate side effects unless idempotency by request ID is enforced. You add dedup tracking before changing code.

### Knowledge persists across sessions

Days later, someone adds a new request path.

The checker fails a liveness monitor because one path never emits response events. The issue is caught before rollout.

### P grounds a design conversation

A proposal merges two states for simplicity.

The monitor suite now cannot distinguish "accepted but not durable" from "durably applied." You keep the intermediate state because that distinction matters for correctness.

### Distillation catches drift

A model says retries stop after three attempts.

Code now retries indefinitely under one branch. `weed` reports the divergence, and you choose whether to update model or code.

## Typical workflow

1. Model behavior in `.p` files (`PSrc/`, `PSpec/`, `PTst/`).
2. Compile with `p compile` (prefer a `.pproj` file).
3. Run `p check -tc <testcase> -s <schedules>`.
4. Iterate model and implementation until monitors and tests pass.

Example `.pproj`:

```xml
<Project>
  <ProjectName>PaymentFlow</ProjectName>
  <InputFiles>
    <PFile>./PSrc/</PFile>
    <PFile>./PSpec/</PFile>
    <PFile>./PTst/</PFile>
  </InputFiles>
  <OutputDir>./PGenerated/</OutputDir>
</Project>
```

## Repository structure

- `SKILL.md`: core P modeling guidance
- `references/language-reference.md`: language and workflow reference for this skill pack
- `references/patterns.md`: reusable modeling patterns
- `references/test-generation.md`: checker-oriented test planning
- `skills/elicit/`: requirements to model workflow
- `skills/distill/`: code to model workflow

## Language governance

This repository keeps a structured review process in [TEAM.md](TEAM.md), with dedicated prompts in [REVIEW.md](REVIEW.md) and [PROPOSE.md](PROPOSE.md), so language guidance changes remain coherent.

## Feedback

If you want behavior or guidance changes, open an issue or submit a PR with:

- concrete modeling pain point
- minimal reproducer
- proposed wording or pattern update

## About the name

`P` is the language name used by the upstream project for formally modeling distributed event-driven systems.

## Copyright & License

See [LICENSE](LICENSE).

## References

- P docs: https://p-org.github.io/P/
- P GitHub repo: https://github.com/p-org/P
- Tutorials: https://p-org.github.io/P/tutsoutline/
