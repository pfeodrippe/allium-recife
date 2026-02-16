---
name: elicit
description: This skill should be used when the user wants to "build a recife model", "elicit requirements", "capture domain behaviour", "specify a feature", or is describing functionality they want to build and needs guidance on extracting an executable model through conversation.
---

# Elicitation

This skill guides you through building Recife model specifications by conversation. The goal is to surface ambiguities and produce a model that captures what the software does without prescribing implementation detail.

The same principles apply to distillation. Whether you are hearing a stakeholder describe a feature or reading code that implements it, the challenge is identical: finding the right level of abstraction.

## Scoping the specification

Before diving into details, establish what you are specifying. Not everything needs to be in one model.

### Questions to ask first

**"What's the boundary of this specification?"** A complete system? A single feature area? One service in a larger system? Be explicit about what is in and out of scope.

**"Are there areas we should deliberately exclude?"** Third-party integrations might be library models. Legacy features might not be worth specifying. Some features might belong in separate models.

**"Is this a new system or does code already exist?"** If code exists, you are doing distillation with elicitation. Existing code constrains what is realistic to specify.

### Documenting scope decisions

Capture scope at the start of every model:

```clojure
(ns interview-scheduling.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

;; Scope: Interview scheduling for the hiring pipeline
;; Includes: candidacy, interview, slot management, invitations, feedback
;; Excludes:
;;   - authentication (use oauth model)
;;   - payments (not applicable)
;;   - reporting dashboards (separate model)
;; Dependencies: user entity defined in core.model.clj

(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)}})
```

Use a namespace declaration as the first form of every model `.clj` file, then record scope comments directly above `global` and the first process definitions.

## Finding the right level of abstraction

The hardest part of specification is choosing what to include and what to leave out. Too concrete and you are specifying implementation. Too abstract and you are not saying anything useful.

### The "Why" test

For every detail, ask: "Why does the stakeholder care about this?"

| Detail | Why? | Include? |
|--------|------|----------|
| "Users log in with Google OAuth" | They need to authenticate | Maybe not, "Users authenticate" might be sufficient |
| "We support Google and Microsoft OAuth" | Users choose their provider | Yes, the choice is domain-level |
| "Sessions expire after 24 hours" | Security/UX decision | Yes, affects user experience |
| "Sessions are stored in Redis" | Performance | No, implementation detail |
| "Passwords must be 12+ characters" | Security policy | Yes, affects users |
| "Passwords are hashed with bcrypt" | Security implementation | No, how not what |

### The "Could it be different?" test

Ask: "Could this be implemented differently while still being the same system?"

- If yes, it is probably an implementation detail. Abstract it away.
- If no, it is probably domain-level. Include it.

Examples:

- "Notifications sent via Slack". Could be email, SMS, etc. Abstract to an emitted notification event in model state.
- "Interviewers must confirm within 3 hours". This specific deadline matters at the domain level. Include the duration.
- "We use PostgreSQL". Could be any database. Do not include.
- "Data is retained for 7 years for compliance". Regulatory requirement. Include.

### The "Template vs Instance" test

Is this a category of thing, or a specific instance?

| Instance (implementation) | Template (domain-level) |
|---------------------------|-------------------------|
| Google OAuth | Authentication provider |
| Slack | Notification channel |
| 15 minutes | Link expiry duration (configurable) |
| Greenhouse ATS | External candidate source |

Sometimes the instance IS the domain concern. "We specifically integrate with Salesforce" might be a competitive feature. "We support exactly these three OAuth providers" might be design scope.

When in doubt, ask the stakeholder: "If we changed this, would it be a different system or just a different implementation?"

### Levels of abstraction

```
Too abstract:          "Users can do things"
                              |
Product level:         "Candidates can accept or decline interview invitations"
                              |
Too concrete:          "Candidates click a button that POST to /api/invitations/:id/accept"
```

**Signs you are too abstract.** The model could describe almost any system. No checkable assertions. Product owner says "but that doesn't capture..."

**Signs you are too concrete.** You are mentioning technologies, frameworks or APIs. You are describing UI elements (buttons, pages, forms). The implementation team says "why are you dictating how we build this?"

### Configuration vs hardcoding

When you encounter a specific value (3 hours, 7 days, etc.), ask:

1. **Is this value a design decision?** Include it.
2. **Might it vary per deployment or customer?** Make it configurable.
3. **Is it arbitrary?** Consider whether to include it at all.

```clojure
;; Hardcoded design decision
(r/defproc invitation-expires
  (fn [{:keys [::invitations :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id invitation]
                 (if (<= (+ (:created-at invitation) (* 7 24 60 60 1000)) now)
                   (assoc-in acc [::invitations invitation-id :status] :expired)
                   acc))
               db
               invitations)))

;; Configurable
(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)}
   ::invitations {}})

(r/defproc invitation-expires
  (fn [{:keys [::invitations ::config :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id invitation]
                 (if (<= (+ (:created-at invitation)
                            (:invitation-expiry-ms config))
                         now)
                   (assoc-in acc [::invitations invitation-id :status] :expired)
                   acc))
               db
               invitations)))
```

### Black boxes

Some logic is important but belongs at a different level:

```clojure
(defn interviewer-matching-suggest
  [{:keys [role-skills interviewer-skills interviewer-availability interviewer-load]}]
  ;; Black box: we know required inputs/outputs, but not internals here.
  (throw (ex-info "Defined in detailed model" {:inputs [role-skills interviewer-skills interviewer-availability interviewer-load]})))

(r/defproc create-suggestion
  (fn [{:keys [::role ::interviewers] :as db}]
    (update db ::suggestions conj
            {:interviewers (interviewer-matching-suggest
                            {:role-skills (:required-skills role)
                             :interviewer-skills (map :skills interviewers)
                             :interviewer-availability (map :availability interviewers)
                             :interviewer-load (map :recent-load interviewers)})})))
```

The model says there is a matching algorithm, that it considers these inputs and that it produces interviewer suggestions. The model does not say how matching works, what weights are used or the specific algorithm.

This is the right level when the algorithm is complex and evolving, when product owners care about inputs and outputs rather than internals, and when a separate detailed model could cover it if needed.

## Elicitation methodology

### Phase 1: Scope definition

**Goal:** Understand what we are specifying and where the boundaries are.

Questions to ask:

1. "What is this system fundamentally about? In one sentence?"
2. "Where does this system start and end? What's in scope vs out?"
3. "Who are the users? Are there different roles?"
4. "What are the main things being managed, the nouns?"
5. "Are there existing systems this integrates with? What do they handle?"

**Outputs:** List of actors and roles. List of core entities. Boundary decisions (what is external). One-sentence description.

**Watch for:** Scope creep ("and it also does X, Y, Z", gently refocus). Assumed knowledge ("obviously it handles auth", make explicit).

### Phase 2: Happy path flow

**Goal:** Trace the main journey from start to finish.

Questions to ask:

1. "Walk me through a typical [X] from start to finish"
2. "What happens first? Then what?"
3. "What triggers this? A user action? Time passing? Something else?"
4. "What changes when that happens? What state is different?"
5. "Who needs to know when this happens? How?"

**Technique:** Follow one entity through its lifecycle.

```
Candidacy:
  pending_scheduling -> scheduling_in_progress -> scheduled ->
  interview_complete -> feedback_collected -> decided
```

**Outputs:** State machines for key entities. Main triggers and their outcomes. Communication touchpoints.

**Watch for:** Jumping to edge cases too early ("but what if...", note it and stay on happy path). Implementation details creeping in ("the API endpoint...", redirect to outcomes).

### Phase 3: Edge cases and errors

**Goal:** Discover what can go wrong and how the system handles it.

Questions to ask:

1. "What if [actor] doesn't respond?"
2. "What if [condition] isn't met when they try?"
3. "What if this happens twice? Or in the wrong order?"
4. "How long should we wait before [action]?"
5. "When should a human be alerted to intervene?"
6. "What if [external system] is unavailable?"

**Technique:** For each transition, ask "what are all the ways this guard could fail?"

**Outputs:** Timeout and deadline rules. Retry and escalation logic. Error states. Recovery paths.

**Watch for:** Infinite loops ("then it retries, then retries again...", need terminal states). Missing escalation, because eventually a human needs to know.

### Phase 4: Refinement

**Goal:** Clean up the model and identify gaps.

Questions to ask:

1. "Looking at [entity], are these states complete? Can it be in any other state?"
2. "Is there anything we haven't covered?"
3. "This process references [X], do we need to define that, or is it external?"
4. "Is this detail essential here, or should it live in a detailed model?"

**Technique:** Read back the model and ask "does this match your mental model?"

**Outputs:** Complete entity/state definitions. Open questions documented. Deferred model components identified. External boundaries confirmed.

## Elicitation principles

### Ask one question at a time

Bad: "What entities do you have, and what states can they be in, and who can modify them?"

Good: "What are the main things this system manages?"
Then: "Let's take [Candidacy]. What states can it be in?"
Then: "Who can change a candidacy's state?"

### Work through implications

When a choice arises, do not just accept the first answer. Explore consequences.

"You said invitations expire after 48 hours. What happens then?"
"And if the candidate still hasn't responded after we retry?"
"What if they never respond, is this candidacy stuck forever?"

This surfaces decisions they have not made yet.

### Distinguish product from implementation

When you hear implementation language, redirect:

| They say | You redirect |
|----------|-------------|
| "The API returns a 404" | "So the user is informed it's not found?" |
| "We store it in Postgres" | "What information is captured?" |
| "The frontend shows a modal" | "The user is prompted to confirm?" |
| "We use a cron job" | "This happens on a schedule, how often?" |

### Surface ambiguity explicitly

Better to record an open question than assume.

"I'm not sure whether declining should return the candidate to the pool or remove them entirely. Let me note that as an open question."

```clojure
(comment
  "Open question: when candidate declines, do they return to pool or exit?")
```

### Use concrete examples

Abstract discussions get stuck. Ground them.

"Let's say Alice is a candidate for the Senior Engineer role. She's been sent an invitation with three slots. Walk me through what happens when she clicks on Tuesday 2pm."

### Iterate willingly

It is normal to revise earlier decisions.

"Earlier we said all admins see all notifications. But now you're describing role-specific dashboards. Should we revisit that?"

### Know when to stop

Not everything needs to be specified now.

"This is getting into how the matching algorithm works. Should we defer that to a detailed model?"

"We've covered the main flow. The reporting dashboard sounds like a separate specification."

## Common elicitation traps

### The "Obviously" trap

When someone says "obviously" or "of course", probe. "You said obviously the admin approves. Is there ever a case where they don't need to? Could this be automated later?"

### The "Edge Case Spiral" trap

Some people want to cover every edge case immediately. "Let's capture that as an open question and stay on the main flow for now. We'll come back to edge cases."

### The "Technical Solution" trap

Engineers especially jump to solutions. "I hear you saying we need real-time updates. At the domain level, what does the user need to see and when?"

### The "Vague Agreement" trap

Do not accept "yes" without specifics. "You said yes, candidates can reschedule. How many times? Is there a limit? What happens after that?"

### The "Missing Actor" trap

Watch for actions without clear actors. "You said 'the slots are released'. Who or what releases them? Is it automatic, or does someone trigger it?"

### The "Equivalent Terms" trap

When you hear two terms for the same concept, from different stakeholders, existing code or related models, stop and resolve it before continuing.

"You said 'Purchase' but earlier we called this an 'Order'. Which term should we use?"

A comment noting that two terms are equivalent is not a resolution. It guarantees both will appear in the implementation. Pick one term, cross-reference related models and update all references. Do not leave the old term anywhere, not even in "see also" notes.

## Elicitation session structure

**Opening (5 min).** Explain Recife briefly: "We're capturing what the software does in an executable model, not prescribing implementation internals." Set expectations: "I'll ask lots of questions, some obvious-seeming." Agree on scope for this session.

**Scope definition (10-15 min).** Identify actors, entities, boundaries. Get the one-sentence description.

**Happy path (20-30 min).** Trace main flow start to finish. Capture states, triggers, outcomes. Note communications.

**Edge cases (15-20 min).** Timeouts and deadlines. Failure modes. Escalation paths.

**Wrap-up (5-10 min).** Read back key decisions. List open questions. Identify next session scope if needed.

**After session.** Write up model draft. Send for review. Note questions for next session.

## References

- [Language reference](../../references/language-reference.md) — full Recife model syntax
- [Recognising library model opportunities](./references/library-spec-signals.md) — signals, questions and decision framework for identifying reusable integration models during elicitation
