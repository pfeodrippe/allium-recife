# Language reference

## File structure

A Recife model file is a Clojure source file (`.clj`) with a namespace, required libraries and model components.

```clojure
(ns interview-scheduling.model
  (:require [clojure.set :as set]
            [recife.core :as r]
            [recife.helpers :as rh]))

;; Scope: interview scheduling for hiring pipeline
;; Includes: candidacy, invitation, slot confirmation, reminders
;; Excludes: authentication, billing

(def global
  {::candidacies {}
   ::invitations {}
   ::slots {}
   ::config {:invitation-expiry-ms (* 7 24 60 60 1000)}
   :clock/now 0
   :mail/outbox []
   :notifications/outbox []})

(r/defproc invitation-expires
  (fn [{:keys [::invitations ::slots :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id invitation]
                 (if (and (= :pending (:status invitation))
                          (<= (:expires-at invitation) now))
                   (let [acc' (assoc-in acc [::invitations invitation-id :status] :expired)]
                     (reduce (fn [x sid] (assoc-in x [::slots sid :status] :available))
                             acc'
                             (:slot-ids invitation)))
                   acc))
               db
               invitations)))

(rh/definvariant invitation-status-valid
  [{:keys [::invitations]}]
  (every? (fn [[_ inv]]
            (contains? #{:pending :accepted :declined :expired}
                       (:status inv)))
          invitations))

(comment
  @(r/run-model global #{invitation-expires invitation-status-valid}))
```

A typical model file contains:

- `ns` declaration
- initial model state (`global`)
- one or more processes (`r/defproc`)
- checks (`rh/definvariant`, `rh/defproperty`, `rh/defaction-property`)
- optional execution block in `comment`

### Formatting

Indentation is conventional Clojure indentation. Keep process functions small and explicit. Prefer one transition concern per process step. Use namespaced keywords for global state and unnamespaced keys only for process-local variables.

### Naming conventions

- **Namespaces**: kebab-case (`interview-scheduling.model`)
- **Vars**: kebab-case (`invitation-expires`, `candidate-accepts`)
- **Global keys**: namespaced keywords (`::invitations`, `:mail/outbox`)
- **Local keys**: short unnamespaced keys (`:pc`, `:self`, `:slot-id`)
- **Statuses/tags**: namespaced or plain keywords (`:pending`, `:accepted`, `:branch`)

---

## Module given

Recife does not have a `given {}` keyword. Equivalent shared context is represented in `global` and per-process local maps.

```clojure
(def global
  {::pipeline {:status :active}
   ::calendar {:available-slot-ids #{:s1 :s2 :s3}}})

(r/defproc choose-slot
  (fn [{:keys [::pipeline ::calendar] :as db}]
    (if (= :active (:status pipeline))
      (assoc db ::last-available (first (:available-slot-ids calendar)))
      db)))
```

Use `:local` with `defproc` when each process instance needs isolated local state.

```clojure
(r/defproc reminder-worker
  {:procs #{:w1 :w2}
   :local {:pc ::scan
           :batch-size 10}}
  {::scan
   (fn [{:keys [:batch-size] :as db}]
     (if (> batch-size 0)
       (r/goto db ::send)
       (r/done db)))

   ::send
   (fn [db]
     (-> db
         (assoc :batch-size 0)
         r/done))})
```

---

## Entities

In Recife, entities are usually map entries under namespaced collections.

```clojure
(def global
  {::candidacies
   {:cand-1 {:candidate-id :u-1
             :role-id :role-1
             :status :pending
             :slot-ids #{:slot-1 :slot-2}}}})
```

### External entities

External systems are represented as inbound/outbound queues and normalized projections.

```clojure
(def global
  {:greenhouse/inbound #{}
   :stripe/events #{}
   ::candidates {}})

(r/defproc import-candidate
  (fn [{:keys [:greenhouse/inbound] :as db}]
    (when-let [event (first inbound)]
      (-> db
          (update :greenhouse/inbound disj event)
          (assoc-in [::candidates (keyword (:id event))]
                    {:name (:name event)
                     :email (:email event)
                     :source :greenhouse})))))
```

### Internal entities

Internal entities are state maps managed by model transitions.

```clojure
(def global
  {::invitations
   {:inv-1 {:candidacy-id :cand-1
            :status :pending
            :slot-ids #{:slot-1 :slot-2}
            :expires-at 1739786400000}}
   ::slots
   {:slot-1 {:status :confirmed}
    :slot-2 {:status :confirmed}}})
```

### Value types

Value types are plain immutable Clojure values (maps, vectors, sets, numbers, strings).

```clojure
(defn time-range [start end]
  {:start start
   :end end
   :duration-ms (- end start)})

(defn location [name tz country]
  {:name name
   :timezone tz
   :country country})
```

### Sum types

Use tagged maps with `:kind` (or `:type`) and branch with `case`/`cond`.

```clojure
(def node-branch {:kind :branch :path "/" :children [:n1 :n2]})
(def node-leaf {:kind :leaf :path "/tmp/a" :data [1 2 3]})

(defn process-node [node]
  (case (:kind node)
    :branch (count (:children node))
    :leaf (count (:data node))
    0))
```

A sum type discipline in Recife is by convention and checks (invariants), not grammar keywords.

### Field types

Common Recife field shapes:

- string: `"abc"`
- integer: `42`
- decimal: `12.5`
- boolean: `true` / `false`
- timestamp/ms: `1739786400000`
- optional: `nil` or value
- set: `#{:a :b}`
- vector/list-like: `[:a :b :c]`
- nested map: `{:status :pending :expires-at 0}`

Represent absence with `nil`.

```clojure
(defn reminded? [feedback-request]
  (some? (:reminded-at feedback-request)))
```

### Relationships

Relationships are modeled by identifiers and lookups.

```clojure
(def global
  {::candidacies {:cand-1 {:slot-ids #{:slot-1 :slot-2}}}
   ::slots {:slot-1 {:status :confirmed}
            :slot-2 {:status :pending}}})

(defn candidacy-slots [db candidacy-id]
  (->> (get-in db [::candidacies candidacy-id :slot-ids])
       (map #(get-in db [::slots %]))))
```

### Projections

A projection is a derived filtered view.

```clojure
(defn confirmed-slots [db candidacy-id]
  (->> (get-in db [::candidacies candidacy-id :slot-ids])
       (map #(vector % (get-in db [::slots %])))
       (filter (fn [[_ slot]] (= :confirmed (:status slot))))
       (map first)
       set))
```

### Derived values

Derived values are plain helper functions over state.

```clojure
(defn invitation-expired? [invitation now]
  (<= (:expires-at invitation) now))

(defn candidacy-ready? [db candidacy-id]
  (>= (count (confirmed-slots db candidacy-id)) 3))
```

---

## Rules

Rules are process transitions (`r/defproc`) plus checks (`rh/*`).

### Rule structure

```clojure
(r/defproc candidate-accepts
  (fn [{:keys [::invitations ::slots :commands/candidate-accepts :clock/now] :as db}]
    (when-let [{:keys [invitation-id slot-id]} (first candidate-accepts)]
      (let [invitation (get invitations invitation-id)]
        (when (and (= :pending (:status invitation))
                   (> (:expires-at invitation) now)
                   (contains? (set (:slot-ids invitation)) slot-id))
          (-> db
              (update :commands/candidate-accepts disj {:invitation-id invitation-id :slot-id slot-id})
              (assoc-in [::invitations invitation-id :status] :accepted)
              (assoc-in [::slots slot-id :status] :booked)))))))
```

### Rule-level iteration

Use `reduce`, `map`, `for`, `doseq`-style pure transformations.

```clojure
(r/defproc expire-old-invitations
  (fn [{:keys [::invitations :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id invitation]
                 (if (and (= :pending (:status invitation))
                          (<= (:expires-at invitation) now))
                   (assoc-in acc [::invitations invitation-id :status] :expired)
                   acc))
               db
               invitations)))
```

### Multiple rules for the same trigger

Model a trigger as an event queue and let multiple processes consume/respond.

```clojure
(def global
  {:auth/events #{}
   :audit/log []
   ::sessions {}})

(r/defproc lock-account-on-failures
  (fn [{:keys [:auth/events] :as db}]
    ...))

(r/defproc audit-auth-events
  (fn [{:keys [:auth/events] :as db}]
    ...))
```

### Trigger types

Recife trigger patterns:

- **External stimulus**: inbound event queue (`:commands/*`, `:webhooks/*`)
- **State transition**: status/value mutation via `assoc-in`/`update`
- **State becomes**: guard checks target state and transitions once
- **Temporal**: compare state timestamps with `:clock/now`
- **Derived condition**: helper predicate in guard/invariant
- **Entity creation**: insert map entry under collection key
- **Chained**: process A writes queue/event consumed by process B

### Preconditions (requires)

Preconditions are `when`/`if` guards in process functions.

```clojure
(when (and (= :pending (:status invitation))
           (> (:expires-at invitation) now)
           (contains? (set (:slot-ids invitation)) slot-id))
  ...)
```

### Local bindings (let)

Use `let` to name intermediate state and keep transitions readable.

```clojure
(let [invitation (get invitations invitation-id)
      other-slot-ids (disj (set (:slot-ids invitation)) slot-id)]
  ...)
```

### Discard bindings

Use `_` for intentionally ignored values.

```clojure
(every? (fn [[_ invitation]]
          (contains? #{:pending :accepted :declined :expired}
                     (:status invitation)))
        invitations)
```

### Postconditions (ensures)

Postconditions are represented by explicit new state values returned by the process.

Common patterns:

- state update: `(assoc-in db [::x id :status] :accepted)`
- entity creation: `(assoc-in db [::y new-id] {:status :new})`
- side-effect intent event: `(update db :mail/outbox conj {...})`
- entity removal: `(update db ::tokens dissoc token-id)`

---

## Expression language

Recife models use Clojure expressions.

### Navigation

```clojure
(get-in db [::invitations invitation-id :status])
(-> db (get-in [::candidacies :cand-1]) :candidate-id)
```

### Join lookups

```clojure
(let [invitation (get-in db [::invitations :inv-1])
      candidacy (get-in db [::candidacies (:candidacy-id invitation)])]
  (:candidate-id candidacy))
```

### Collection operations

```clojure
(count slot-ids)
(contains? slot-ids :slot-2)
(some #(= :confirmed (:status %)) (vals slots))
(every? #(contains? #{:pending :accepted} (:status %)) (vals invitations))
(set/union #{:a} #{:b})
(set/difference #{:a :b} #{:b})
```

### Comparisons

```clojure
(= :pending status)
(not= :expired status)
(< attempts max-attempts)
(<= expires-at now)
```

### Arithmetic

```clojure
(+ now (* 7 24 60 60 1000))
(- current-limit consumed)
(>= usage quota)
```

### Boolean logic

```clojure
(and active? paid?)
(or admin? support?)
(not suspended?)
```

### Conditional expressions

```clojure
(if overdue?
  (assoc db :status :expired)
  db)

(cond
  (= :pending status) :open
  (= :accepted status) :closed
  :else :unknown)
```

### Existence

```clojure
(contains? invitations invitation-id)
(some? (get-in db [::users user-id]))
(nil? (get-in db [::feedback-requests req-id :reminded-at]))
```

### Literals

```clojure
"text"
42
12.5
true
false
nil
:pending
#{:a :b}
{:status :pending}
```

### Black box functions

Use pure helper functions to represent complex logic without expanding internals.

```clojure
(defn interviewer-matching-suggest [_db _request]
  ;; Deferred detailed model; returns candidate interviewer ids.
  #{:i-1 :i-3})
```

### The `with` and `where` keywords

Recife has no special `with`/`where` syntax. Use Clojure `let`, `filter`, and helper functions.

```clojure
(let [confirmed (filter (fn [[_ slot]] (= :confirmed (:status slot))) slots)]
  (into #{} (map first confirmed)))
```

### Entity collections

Entity collections are usually maps keyed by ids.

```clojure
(def global
  {::users {:u-1 {:status :active}
            :u-2 {:status :locked}}
   ::subscriptions {:s-1 {:user-id :u-1 :status :active}}})
```

---

## Deferred specifications

Deferred logic should be represented as explicit placeholders and linked to dedicated model files.

```clojure
(defn escalation-policy-at-level
  [_db _level]
  ;; see: detailed/escalation_policy.clj
  (throw (ex-info "Deferred model component" {})))
```

Unlike opaque helper functions, deferred model components are expected to exist as separate model modules and be versioned with the main model.

## Open questions

Track unresolved decisions close to the model.

```clojure
(comment
  "Open question: should support agents have read-only billing access?"
  "Open question: trial conversion grace period duration")
```

## Config

Keep configurable parameters in explicit config maps.

```clojure
(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)
             :max-login-attempts 5
             :digest-window-ms (* 24 60 60 1000)}})
```

Use config values in process guards and updates.

```clojure
(<= (+ created-at (get-in db [::config :invitation-expiry-ms])) now)
```

## Defaults

Model defaults are initial state values in `global`.

```clojure
(def global
  {::roles {:viewer {:permissions #{:documents.read}}
            :editor {:permissions #{:documents.read :documents.write}}}})
```

## Modular specifications

Recife models are composed with namespaces and required modules.

### Namespaces

```clojure
(ns app.model.auth
  (:require [recife.core :as r]
            [recife.helpers :as rh]))
```

Use one namespace per cohesive model concern.

### Using other specs

Compose models by requiring namespaces and including their components in `run-model`.

```clojure
(ns app.model.main
  (:require [app.model.auth :as auth]
            [app.model.billing :as billing]
            [recife.core :as r]))

(comment
  @(r/run-model auth/global
                #{auth/login-flow
                  billing/subscription-flow
                  auth/no-duplicate-sessions}))
```

### Referencing external entities and triggers

External boundaries are normalized queues or maps, then referenced by local processes.

```clojure
(def global
  {:stripe/events #{}
   ::subscriptions {}})

(r/defproc apply-stripe-event
  (fn [{:keys [:stripe/events] :as db}]
    ...))
```

### Responding to external triggers

Use inbound queues and deterministic consumption patterns.

```clojure
(r/defproc consume-webhook
  (fn [{:keys [:webhook/events] :as db}]
    (when-let [event (first events)]
      (-> db
          (update :webhook/events disj event)
          (update :audit/log conj {:event event})))))
```

### Configuration

Each module may carry local config maps (`::config` key) or receive shared config in combined global state.

### Breaking changes

Treat model schema/key changes as breaking when:

- component names change (`defproc`/check var names)
- state key paths change (`::x` to `::y`)
- event shapes change (`{:type ...}` structure)

Provide migration notes and update composed runners.

### Local specs

Local modules should be normal Clojure files in the repo (`app/model/*.clj`) and referenced by namespace.

---

## Surfaces

Recife has no dedicated `surface` keyword. Boundary contracts are represented with explicit state slices, operation queues and invariants.

```clojure
(def global
  {::dashboard {:viewer-id :i-1
                :visible-fields #{:slot :status}
                :allowed-actions #{:confirm-slot}}
   :ui/commands #{}})
```

### Actor declarations

Actors are modeled as entities or role tags in state.

```clojure
(def global
  {::users {:u-1 {:role :interviewer}
            :u-2 {:role :admin}}})

(defn actor-can-confirm-slot? [db user-id]
  (= :interviewer (get-in db [::users user-id :role])))
```

### Surface structure

A practical surface contract in Recife usually includes:

- visible data projection (`::dashboard` map)
- allowed operations (`:allowed-actions`)
- command queue for requested operations (`:ui/commands`)
- invariants enforcing role-based visibility/permissions

### Examples

```clojure
(r/defproc confirm-slot
  (fn [{:keys [::users :ui/commands ::slots] :as db}]
    (when-let [{:keys [user-id slot-id]} (first commands)]
      (if (= :interviewer (get-in users [user-id :role]))
        (-> db
            (update :ui/commands disj {:user-id user-id :slot-id slot-id})
            (assoc-in [::slots slot-id :status] :confirmed))
        db))))

(rh/definvariant only-interviewer-confirms
  [{:keys [::users ::slots :audit/confirmations]}]
  (every? (fn [{:keys [user-id]}]
            (= :interviewer (get-in users [user-id :role])))
          confirmations))
```

---

## Validation rules

A robust Recife model should satisfy:

1. Every `defproc` returns either `nil` (no transition) or a valid state map.
2. Global keys are namespaced; local process keys are unnamespaced.
3. Invariants hold across all reachable states.
4. Temporal properties are meaningful and checkable.
5. Event queues are consumed or intentionally retained.
6. State transitions are monotonic where required by domain policy.
7. Status/tag sets are constrained via invariants.
8. No process depends on hidden mutable external state.
9. Non-deterministic params (when used) are finite sets/colls or functions returning finite collections.
10. Model time (`:clock/now`) is explicit in temporal logic.

Recommended workflow:

```clojure
(comment
  (def components #{process-a process-b invariant-a property-a})
  (def result @(r/run-model global components {:workers 4}))
  result)
```

---

## Anti-patterns

Avoid:

- embedding HTTP/database framework details in model state semantics
- using model keys that mirror table/ORM internals without domain meaning
- hiding important transitions inside opaque helper calls with no checks
- overloading one process with unrelated responsibilities
- adding non-determinism that is infinite or not controlled
- using unbounded collections without constraining behavior via invariants/properties
- mixing command representation formats in one queue
- writing checks that merely restate code mechanics instead of domain guarantees

Prefer:

- small composable processes
- explicit event queues
- explicit invariants for safety
- explicit temporal properties for liveness
- clear namespace/module boundaries

---

## Glossary

- **Model state**: the map processed by Recife (`db`), containing global and local process data.
- **Process (`defproc`)**: a transition producer; returns next state or `nil`.
- **Step map**: multi-step process definition keyed by step ids, using `:pc` and `r/goto`/`r/done`.
- **Invariant**: safety property that must always hold (`rh/definvariant`).
- **Temporal property**: liveness/temporal expectation (`rh/defproperty`).
- **Action property**: relation between consecutive states (`rh/defaction-property`).
- **Fairness**: scheduling assumptions for process execution (`rh/deffairness` or `^:fair`).
- **Global key**: namespaced keyword persisted in global model state.
- **Local key**: unnamespaced keyword scoped to process instance local state.
- **Command queue**: collection of requested external actions/events consumed by processes.
- **Projection**: helper-derived view over entity collections.
- **Deferred model**: referenced logic modeled in another file/module.
