# Recife

*Velocity through executable clarity*

---

A Clojure model checking workflow for sharpening intent alongside implementation.

## Get started

**Claude Code** (via plugin marketplace):

```
/plugin marketplace add juxt/claude-plugins
/plugin install recife
```

**Cursor, Windsurf, Copilot, Aider, Continue and other tools:**

```
npx skills add pfeodrippe/recife-skill
```

Once installed, type `/recife` to get started. The skill examines your project and offers to distill from existing code or build a new model through conversation. You can also jump straight to a specific mode:

- `/recife:elicit` — build a model through structured conversation with stakeholders
- `/recife:distill` — extract a model from existing code

Jump to what [Recife looks like in practice](#what-this-looks-like-in-practice).

## The problem with conversational context

- Within a session, meaning drifts: by prompt ten or twenty, the model is pattern-matching on its own outputs rather than the original intent.
- Across sessions, knowledge evaporates: assumptions and constraints disappear when the chat ends.

Recife gives behavioural intent a durable, executable form that does not drift with the conversation and persists across sessions.

## Why not just point the LLM at the code?

Modern LLMs navigate codebases effectively. The limitation appears when you need to distinguish what the code *does* from what it *should do*. Code captures implementation, including bugs and expedient decisions. The model treats all of it as intended behaviour.

Precise prompting helps, but precise prompting still means specifying intent: which behaviours are deliberate, which constraints must be preserved. You end up writing descriptions of intent distributed across prompts. Recife captures this intent as executable models and checks.

## Why not capture requirements in markdown?

Markdown provides no machinery for checking contradictions over state transitions. You can write “users must be authenticated” in one section and “guest checkout is supported” in another without the format highlighting tension.

Recife models make those contradictions checkable via invariants, action properties and temporal properties. You do not rely on prose interpretation alone.

## Iterating on specifications

The model and the code evolve together. Writing and refining behavioural models alongside implementation sharpens understanding of both the problem and the solution.

Two processes feed this growth: **elicitation** works forward from intent through structured conversations with stakeholders, while **distillation** works backward from implementation to capture what the system actually does, including behaviour never explicitly decided.

See the [elicitation guide](skills/elicit/SKILL.md) and the [distillation guide](skills/distill/SKILL.md).

## On single sources of truth

A common objection is that maintaining behavioural models alongside code violates single-source-of-truth principles. In practice, code captures both intentional and accidental behaviour. You still need an explicit behavioural layer to state what should hold.

Recife applies the same pattern as tests and type systems: code expresses *how*; models and properties express *what must always hold* and *what must eventually happen*.

## What Recife captures

Recife provides executable Clojure syntax for describing transitions, constraints and temporal expectations.

```clojure
(ns model.password-reset
  (:require [recife.core :as r]))

(def global
  {::config {:reset-token-expiry-ms (* 24 60 60 1000)}
   ::users {:u-1 {:email "ana@example.com" :status :active}
            :u-2 {:email "bob@example.com" :status :locked}}
   ::reset-tokens {}
   :auth/request-password-reset #{}
   :mail/outbox []
   :clock/now 0})

(r/defproc request-password-reset
  (fn [{:keys [:auth/request-password-reset ::users ::reset-tokens ::config :clock/now] :as db}]
    (when-let [{:keys [email]} (first request-password-reset)]
      (when-let [[user-id user] (first (filter (fn [[_ u]] (= email (:email u))) users))]
        (when (contains? #{:active :locked} (:status user))
          (let [token-id (keyword (str "token-" (inc (count reset-tokens))))
                db' (-> db
                        (update :auth/request-password-reset disj {:email email})
                        (update ::reset-tokens
                                (fn [tokens]
                                  (into {}
                                        (map (fn [[tid token]]
                                               [tid (if (and (= user-id (:user-id token))
                                                             (= :pending (:status token)))
                                                      (assoc token :status :expired)
                                                      token)]))
                                        tokens)))
                        (assoc-in [::reset-tokens token-id]
                                  {:user-id user-id
                                   :created-at now
                                   :expires-at (+ now (:reset-token-expiry-ms config))
                                   :status :pending}))]
            (update db' :mail/outbox conj
                    {:to (:email user)
                     :template :password-reset
                     :data {:token-id token-id}})))))))
```

This model captures observable behaviour without encoding database or transport details.

The same syntax works for operational policy and reliability controls:

```clojure
(ns model.circuit-breaker
  (:require [recife.core :as r]))

(def global
  {::circuit-breaker {:service :external-service
                      :status :closed
                      :opened-at nil
                      :failures []}
   ::config {:failure-threshold 0.5
             :failure-window-ms 30000
             :window-sample-size 20
             :recovery-timeout-ms 10000}
   :clock/now 0})

(defn recent-failures [db]
  (let [cutoff (- (:clock/now db) (get-in db [::config :failure-window-ms]))]
    (filter #(> (:occurred-at %) cutoff)
            (get-in db [::circuit-breaker :failures]))))

(defn failure-rate [db]
  (/ (count (recent-failures db))
     (double (get-in db [::config :window-sample-size]))))

(defn tripped? [db]
  (>= (failure-rate db) (get-in db [::config :failure-threshold])))
(r/defproc circuit-opens
  (fn [{:keys [::circuit-breaker :clock/now] :as db}]
    (when (and (tripped? db)
               (= :closed (:status circuit-breaker)))
      (-> db
          (assoc-in [::circuit-breaker :status] :open)
          (assoc-in [::circuit-breaker :opened-at] now)))))
(r/defproc circuit-probes
  (fn [{:keys [::circuit-breaker ::config :clock/now] :as db}]
    (when (and (= :open (:status circuit-breaker))
               (<= (+ (:opened-at circuit-breaker)
                      (:recovery-timeout-ms config))
                   now))
      (assoc-in db [::circuit-breaker :status] :half-open))))
```

```clojure
(ns model.incident-escalation
  (:require [recife.core :as r]))

(def global
  {::incident {:status :open
               :declared-at 0
               :sla-target-ms 300000
               :escalation-level 0}
   ::config {:exec-notify-threshold 2}
   :clock/now 0
   :pager/requests []
   :exec/briefings []})

(defn escalation-policy-at-level [_level]
  :oncall-team)
(r/defproc escalate-incident
  (fn [{:keys [::incident ::config :clock/now] :as db}]
    (when (and (contains? #{:open :investigating} (:status incident))
               (<= (+ (:declared-at incident) (:sla-target-ms incident)) now))
      (let [level (inc (:escalation-level incident))]
        (cond-> (-> db
                    (assoc-in [::incident :escalation-level] level)
                    (update :pager/requests conj
                            {:team (escalation-policy-at-level level)
                             :priority :immediate}))
          (>= level (:exec-notify-threshold config))
          (update :exec/briefings conj {:incident :primary :level level}))))))
```

The [language reference](references/language-reference.md) covers processes, non-determinism, invariants, temporal properties, fairness and model composition.

### A language with an executable checker

Recife is not just descriptive. It runs models and checks properties. In an LLM workflow, this gives an explicit behavioural artifact that can be validated continuously instead of remaining an unverified narrative.

## What this looks like in practice

### Recife surfaces implications you missed

A customer account was suspended for a policy violation. The team asks to allow password resets anyway.

---

> ***> Can you enable password reset for suspended users?***
>
> ✻ Your Recife model only allows resets for statuses `:active` and `:locked`. Enabling `:suspended` would bypass the reinstatement flow currently captured as a separate transition.
>
> ✻ Do we want to reinstate through admin approval first, then allow reset, so the audit step remains required?
>
> ***> Yes, keep admin reinstatement mandatory.***

---

Without a model, the assistant might relax checks in a way that bypasses controls.

### Knowledge persists across sessions

A developer returns days later to add a free trial.

---

> ***> Add a free first month for new subscriptions.***
>
> ✻ Your model currently requires payment capture before `:subscription-active`. A free month needs either a zero-value invoice path or a trial status with explicit conversion rules.
>
> ✻ Should trial activation still collect payment method upfront?
>
> ***> Yes, capture payment method upfront and emit a zero-value invoice.***

---

The previously captured behaviour still shapes new implementation work.

### Recife grounds a design conversation

---

> ***> Add an endpoint so admins can view any user's payment history.***
>
> ✻ Your model does not grant blanket admin access. Access is scope-based. Should this require `:billing-read`?
>
> ***> Support also needs access for disputes.***
>
> ✻ Then splitting `:billing` into `:billing-read` and `:billing-write` keeps support read-only and finance read/write.
>
> ***> Do that. Update the model first.***

---

The model reframes a simple request into a controlled policy decision.

### Distillation catches drift

---

> ***> Does our auth implementation still match the Recife model?***
>
> ✻ Divergence found. The model locks accounts after 5 failures, but `auth/login.clj` is effectively locking after 3 and does not reset failed attempts on success.
>
> ***> Model is authoritative. Fix implementation.***

---

Recife turns “looks right” into a concrete behavioural check.

## Language governance

Every change to this Recife skill is debated by a [nine-member review panel](TEAM.md) before adoption. Each panellist represents a distinct design priority: simplicity, machine reasoning, composability, readability, formal rigour, domain modelling, developer experience, creative ambition and backward compatibility.

The panel operates in two modes. [Reviews](REVIEW.md) evaluate fixes to rough edges in existing guidance. [Proposals](PROPOSE.md) evaluate new features and ambitious changes. Both follow the same debate protocol: present, respond, rebut, synthesise, verdict.

## Feedback

Successes, rough edges and missing capabilities are all useful feedback. Please [raise an issue](https://github.com/pfeodrippe/recife/issues).

## About the name

Recife is a city in Pernambuco, Brazil. The name reflects a practical, grounded modelling approach: executable behaviour that stays close to real implementation constraints.

---

## Copyright & License

The MIT License (MIT)

Copyright © 2026 JUXT Ltd.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
