---
name: recife
description: Clojure model checking for sharpening intent alongside implementation. Velocity through executable models.
version: 1
auto_trigger:
  - file_patterns: ["**/*.clj"]
  - keywords: ["recife", "recife model", "model checker", ".clj model"]
---

# Recife

Recife is a Clojure model checking library for capturing and validating system behaviour at the domain level. It sits between informal feature descriptions and implementation, providing executable models with invariants and temporal properties.

The name comes from Recife, a city in Brazil, and reflects the project's Clojure-first approach to practical model checking.

Key principles:

- Describes observable behaviour, not infrastructure details
- Captures domain logic as executable Clojure model processes
- Checks safety (invariants) and liveness (temporal properties)
- Forces ambiguities into the open before implementation
- Implementation-agnostic: the same model can validate many implementations

Recife does NOT prescribe framework choices, database schemas, API shapes or UI layouts, unless those are domain-level constraints that should be modelled explicitly.

## Routing table

| Task | Skill | When |
|------|-------|------|
| Writing or reading Recife `.clj` model files | this skill | You need Recife syntax and structure |
| Building a model through conversation | `elicit` | User describes behaviour they want to model |
| Extracting a model from existing code | `distill` | User has implementation code and wants a Recife model |

## Quick syntax summary

### Entity

In Recife, domain entities are usually modelled as maps in global state.

```clojure
(def global
  {::candidacies
   {:cand-1 {:candidate-id :candidate-1
          :role-id :role-1
          :status :pending
          :retry-count 0
          :invitation-id :inv-1
          :slot-ids #{:slot-1 :slot-2 :slot-3}}}
   ::invitations
   {:inv-1 {:candidacy-id :cand-1
            :expires-at 1739786400000}}
   ::interview-slots
   {:slot-1 {:candidacy-id :cand-1 :status :confirmed}
    :slot-2 {:candidacy-id :cand-1 :status :pending}
    :slot-3 {:candidacy-id :cand-1 :status :confirmed}}})

(defn confirmed-slots [db candidacy-id]
  (->> (get-in db [::candidacies candidacy-id :slot-ids])
       (filter #(= :confirmed (get-in db [::interview-slots % :status])))
       set))

(defn candidacy-ready? [db candidacy-id]
  (>= (count (confirmed-slots db candidacy-id)) 3))
```

### External entity

External systems are represented as namespaced keys and process boundaries.

```clojure
(def global
  {::roles {:role-1 {:title "Senior Engineer"
                     :required-skills #{:clojure :distributed-systems}
                     :location {:name "Remote" :timezone "UTC"}}}
   :oauth/requests #{}
   :oauth/responses #{}
   :calendar/requests #{}
   :calendar/responses #{}})
```

### Value type

Value types are plain immutable Clojure values.

```clojure
(defn time-range [start end]
  {:start start
   :end end
   :duration (- end start)})
```

### Sum type

Use tagged maps (`:kind` / `:type`) for variants.

```clojure
(def node-branch {:path "/" :kind :Branch :children [:node-1 :node-2]})
(def node-leaf {:path "/tmp/a" :kind :Leaf :data [1 2 3] :log [7 8]})

(defn process-node [node]
  (case (:kind node)
    :Branch {:children (:children node)}
    :Leaf {:combined (concat (:data node) (:log node))}))
```

Use `case`, `cond`, or predicate guards in processes/invariants to narrow variants.

### Module given

The model `global` map is the shared module context.

```clojure
(def global
  {::pipeline {:status :active}
   ::calendar {:available-slots #{:s1 :s2}}})
```

### Rule

Rules are process steps (`r/defproc`) that transition state.

```clojure
(ns model.invitation
  (:require [recife.core :as r]))

(r/defproc invitation-expires
  (fn [{:keys [::invitations ::interview-slots :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id invitation]
                 (let [remaining-slot-ids (filter #(not= :cancelled
                                                        (get-in acc [::interview-slots % :status]))
                                                  (:proposed-slot-ids invitation))]
                   (if (and (= :pending (:status invitation))
                            (<= (:expires-at invitation) now))
                     (let [acc' (assoc-in acc [::invitations invitation-id :status] :expired)]
                       (reduce (fn [x sid]
                                 (assoc-in x [::interview-slots sid :status] :cancelled))
                               acc'
                               remaining-slot-ids))
                     acc)))
               db
               invitations)))
```

### Trigger types

- **External stimulus**: process consumes queued events (`:api/reqs`, `:webhook/reqs`)
- **State transition**: process updates status/fields (`assoc-in`, `update-in`)
- **State becomes**: guard matches target value (`when (= :scheduled status) ...`)
- **Temporal**: guard compares time fields against `:now`
- **Derived condition**: invariant/property observes computed condition
- **Entity creation**: process inserts new map entry (`assoc-in`/`update`)
- **Chained**: one process writes data consumed by another process

### Rule-level iteration

Use regular Clojure iteration in process bodies.

```clojure
(r/defproc process-digests
  (fn [{:keys [::digest-schedule ::users :clock/now] :as db}]
    (if (<= (:next-run-at digest-schedule) now)
      (reduce (fn [acc [user-id user]]
                (if (get-in user [:notification-setting :digest-enabled])
                  (update acc :digest/batches conj {:user-id user-id
                                                    :created-at now})
                  acc))
              db
              users)
      db)))
```

### Ensures patterns

Recife outcomes are state transitions expressed in Clojure:

- **State changes**: `assoc`, `assoc-in`, `update`, `update-in`
- **Entity creation**: add map entries / append events to collections
- **Trigger emission**: append to event queues for other processes
- **Entity removal**: `dissoc` / `update` with `disj` or filtered collections

### Surface

Boundary contracts are modelled explicitly as operations and visibility rules in state.

```clojure
(def global
  {::slot-confirmations
   {:sc-1 {:interviewer-id :i-1
           :slot {:id :s-1 :time 1739786400000}
           :status :pending
           :interview-id :int-1}}
   :surfaces/interviewer-dashboard
   {:facing {:viewer-id :i-1}
    :context {:assignment-id :sc-1}
    :exposes #{:slot.time :status}
    :provides [{:op :InterviewerConfirmsSlot
                :args [:i-1 :s-1]
                :when (fn [db]
                        (= :pending (get-in db [::slot-confirmations :sc-1 :status])))}]
    :related [{:surface :InterviewDetail
               :args [:int-1]
               :when (fn [db]
                       (some? (get-in db [::slot-confirmations :sc-1 :interview-id])))}]}})
```

### Surface-to-implementation contract

Model the exact fields/actions each boundary exposes, then verify implementation traces against those expectations.

### Expressions

Use plain Clojure expressions (`get-in`, `assoc-in`, `update`, `filter`, `some`, `every?`, `contains?`, arithmetic, boolean logic) inside processes, invariants and temporal properties.

### Modular specs

Split models into namespaces and require them from composition namespaces.

```clojure
(ns app.model
  (:require [recife.core :as r]))

(def dependencies
  {:oauth {:source "github.com/recife-models/google-oauth/abc123def"}
   :candidacy {:source "./candidacy.clj"}})

(def qualified-refs
  {:session [:oauth/session]
   :candidacy [:candidacy/candidacy]})
```

Local references use relative Clojure namespaces/files, e.g. `app/model/scheduling.clj`.

### Config

Use explicit config maps in the model state.

```clojure
(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)
             :max-login-attempts 5}})
```

### Defaults

Provide default model entities directly in `global`.

```clojure
(def global
  {::roles {:viewer {:name "viewer"
                     :permissions #{:documents.read}}}})
```

### Deferred specs

Represent deferred logic as placeholder functions/processes and annotate TODO links.

```clojure
(defn interviewer-matching-suggest
  [_db]
  ;; see: detailed/interviewer_matching.clj
  (throw (ex-info "Deferred model" {})))
```

### Open questions

Track unresolved design choices as explicit comments near the model.

```clojure
(comment
  "Open question: should admins be assigned to specific roles?")
```

## References

- [Language reference](./references/language-reference.md) — full Recife model syntax and conventions
- [Test generation](./references/test-generation.md) — deriving tests from Recife models
- [Patterns](./references/patterns.md) — worked Recife patterns for auth, RBAC, invitations, soft delete, notifications, quotas, comments and integrations
