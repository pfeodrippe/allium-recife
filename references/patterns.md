# Complete patterns

This library contains reusable Recife model patterns for common SaaS scenarios. Each pattern demonstrates practical Clojure model structure and can be adapted to your domain.

## Pattern 1: Password Authentication with Reset

File: `password-auth.model.clj`

```clojure
(ns patterns.password-auth.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::users {:u-1 {:email "ana@example.com"
                  :status :active
                  :failed-login-attempts 0}}
   ::reset-tokens {}
   ::config {:max-login-attempts 5
             :reset-expiry-ms (* 24 60 60 1000)}
   :auth/commands #{}
   :mail/outbox []
   :clock/now 0})

(r/defproc request-password-reset
  (fn [{:keys [:auth/commands ::users ::reset-tokens ::config :clock/now] :as db}]
    (when-let [{:keys [email]} (first commands)]
      (if-let [[user-id user] (first (filter (fn [[_ u]] (= email (:email u))) users))]
        (let [token-id (keyword (str "t-" (inc (count reset-tokens))))]
          (-> db
              (update :auth/commands disj {:email email})
              (assoc-in [::reset-tokens token-id]
                        {:user-id user-id
                         :status :pending
                         :expires-at (+ now (:reset-expiry-ms config))})
              (update :mail/outbox conj {:to email :template :password-reset :token-id token-id})))
        db))))

(rh/definvariant reset-token-user-exists
  [{:keys [::reset-tokens ::users]}]
  (every? (fn [[_ token]] (contains? users (:user-id token))) reset-tokens))
```

## Pattern 2: Role-Based Access Control (RBAC)

File: `rbac.model.clj`

```clojure
(ns patterns.rbac.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::roles {:admin {:permissions #{:billing.read :billing.write :users.read}}
            :support {:permissions #{:billing.read :users.read}}
            :viewer {:permissions #{:users.read}}}
   ::users {:u-1 {:role :admin}
            :u-2 {:role :support}}
   :access/requests #{}
   :access/results #{}})

(defn allowed? [db user-id permission]
  (contains? (get-in db [::roles (get-in db [::users user-id :role]) :permissions])
             permission))

(r/defproc authorize
  (fn [{:keys [:access/requests] :as db}]
    (when-let [{:keys [user-id permission]} (first requests)]
      (-> db
          (update :access/requests disj {:user-id user-id :permission permission})
          (update :access/results conj {:user-id user-id
                                        :permission permission
                                        :allowed? (allowed? db user-id permission)})))))

(rh/definvariant role-exists-for-user
  [{:keys [::users ::roles]}]
  (every? (fn [[_ user]] (contains? roles (:role user))) users))
```

## Pattern 3: Invitation to Resource

File: `resource-invitation.model.clj`

```clojure
(ns patterns.resource-invitation.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::invitations {}
   ::memberships {}
   :resource/commands #{}
   :clock/now 0
   ::config {:invitation-expiry-ms (* 7 24 60 60 1000)}})

(r/defproc create-invitation
  (fn [{:keys [:resource/commands ::invitations ::config :clock/now] :as db}]
    (when-let [{:keys [resource-id email]} (first commands)]
      (let [inv-id (keyword (str "inv-" (inc (count invitations))))]
        (-> db
            (update :resource/commands disj {:resource-id resource-id :email email})
            (assoc-in [::invitations inv-id]
                      {:resource-id resource-id
                       :email email
                       :status :pending
                       :expires-at (+ now (:invitation-expiry-ms config))}))))))

(r/defproc accept-invitation
  (fn [{:keys [:resource/commands ::invitations ::memberships :clock/now] :as db}]
    (when-let [{:keys [invitation-id user-id]} (first commands)]
      (let [invitation (get invitations invitation-id)]
        (if (and invitation
                 (= :pending (:status invitation))
                 (> (:expires-at invitation) now))
          (-> db
              (update :resource/commands disj {:invitation-id invitation-id :user-id user-id})
              (assoc-in [::invitations invitation-id :status] :accepted)
              (assoc-in [::memberships [(:resource-id invitation) user-id]] {:role :member}))
          db)))))

(rh/definvariant accepted-has-membership
  [{:keys [::invitations ::memberships]}]
  (every? (fn [[_ inv]]
            (if (= :accepted (:status inv))
              (some (fn [[[resource-id _] _]] (= resource-id (:resource-id inv))) memberships)
              true))
          invitations))
```

## Pattern 4: Soft Delete & Restore

File: `soft-delete.model.clj`

```clojure
(ns patterns.soft-delete.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::documents {:d-1 {:status :active}
                :d-2 {:status :active}}
   :document/commands #{}})

(r/defproc soft-delete
  (fn [{:keys [:document/commands] :as db}]
    (when-let [{:keys [document-id]} (first commands)]
      (-> db
          (update :document/commands disj {:document-id document-id})
          (assoc-in [::documents document-id :status] :deleted)))))

(r/defproc restore
  (fn [{:keys [:document/commands] :as db}]
    (when-let [{:keys [restore-id]} (first commands)]
      (-> db
          (update :document/commands disj {:restore-id restore-id})
          (assoc-in [::documents restore-id :status] :active)))))

(rh/definvariant no-unknown-status
  [{:keys [::documents]}]
  (every? (fn [[_ doc]] (contains? #{:active :deleted} (:status doc))) documents))
```

## Pattern 5: Notification Preferences & Digests

File: `notifications.model.clj`

```clojure
(ns patterns.notifications.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::users {:u-1 {:digest-enabled? true :timezone "UTC"}
            :u-2 {:digest-enabled? false :timezone "UTC"}}
   ::pending-notifications []
   ::digest-batches []
   :clock/now 0})

(r/defproc batch-digest
  (fn [{:keys [::users ::pending-notifications ::digest-batches] :as db}]
    (reduce-kv (fn [acc user-id user]
                 (if (:digest-enabled? user)
                   (update acc ::digest-batches conj
                           {:user-id user-id
                            :notifications (filter #(= user-id (:user-id %)) pending-notifications)})
                   acc))
               db
               users)))

(rh/definvariant digest-batch-user-exists
  [{:keys [::digest-batches ::users]}]
  (every? (fn [{:keys [user-id]}] (contains? users user-id)) digest-batches))
```

## Pattern 6: Usage Limits & Quotas

File: `usage-limits.model.clj`

```clojure
(ns patterns.usage-limits.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::accounts {:a-1 {:plan :pro
                     :usage {:projects 2}
                     :quota {:projects 3}}}
   :quota/commands #{}})

(r/defproc create-project
  (fn [{:keys [:quota/commands] :as db}]
    (when-let [{:keys [account-id]} (first commands)]
      (let [used (get-in db [::accounts account-id :usage :projects])
            limit (get-in db [::accounts account-id :quota :projects])]
        (if (< used limit)
          (-> db
              (update :quota/commands disj {:account-id account-id})
              (update-in [::accounts account-id :usage :projects] inc))
          db)))))

(rh/definvariant usage-never-exceeds-quota
  [{:keys [::accounts]}]
  (every? (fn [[_ account]]
            (<= (get-in account [:usage :projects])
                (get-in account [:quota :projects])))
          accounts))
```

## Pattern 7: Comments with Mentions

File: `comments.model.clj`

```clojure
(ns patterns.comments.model
  (:require [clojure.string :as str]
            [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::comments {}
   :comment/commands #{}
   :notifications/outbox []})

(defn extract-mentions [text]
  (->> (str/split text #"\\s+")
       (filter #(str/starts-with? % "@"))
       (map #(keyword (subs % 1)))
       set))

(r/defproc create-comment
  (fn [{:keys [:comment/commands ::comments :notifications/outbox] :as db}]
    (when-let [{:keys [comment-id author-id text]} (first commands)]
      (let [mentions (extract-mentions text)]
        (-> db
            (update :comment/commands disj {:comment-id comment-id :author-id author-id :text text})
            (assoc-in [::comments comment-id] {:author-id author-id :text text :mentions mentions})
            (update :notifications/outbox into
                    (map (fn [mentioned-id]
                           {:to mentioned-id :type :mention :comment-id comment-id})
                         mentions)))))))

(rh/definvariant no-self-mention-notification
  [{:keys [::comments :notifications/outbox]}]
  (every? (fn [{:keys [to comment-id]}]
            (not= to (get-in comments [comment-id :author-id])))
          outbox))
```

## Pattern 8: Integrating Library Specs

In Recife, this means composing reusable library model namespaces with application model namespaces.

### Example: OAuth Authentication

Files:

- `oauth2/model.clj` (library model)
- `app-auth/model.clj` (application model)

```clojure
(ns app-auth.model
  (:require [oauth2.model :as oauth]
            [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  (merge oauth/global
         {::users {}
          :app/auth-events #{}}))

(r/defproc create-user-from-oauth
  (fn [{:keys [:oauth/events] :as db}]
    (if-let [event (first (filter #(= :authentication-succeeded (:type %)) events))]
      (-> db
          (update :oauth/events disj event)
          (assoc-in [::users (keyword (:subject event))]
                    {:email (:email event)
                     :status :active}))
      db)))

(rh/definvariant oauth-user-has-email
  [{:keys [::users]}]
  (every? (fn [[_ user]] (string? (:email user))) users))
```

### Example: Payment Processing

Files:

- `stripe-billing/model.clj` (library model)
- `billing/model.clj` (application model)

```clojure
(ns billing.model
  (:require [stripe-billing.model :as stripe]
            [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  (merge stripe/global
         {::subscriptions {:sub-1 {:status :trial}}
          :billing/events #{}}))

(r/defproc activate-subscription-on-payment
  (fn [{:keys [:stripe/events] :as db}]
    (if-let [event (first (filter #(= :invoice-paid (:type %)) events))]
      (-> db
          (update :stripe/events disj event)
          (assoc-in [::subscriptions (keyword (:subscription-id event)) :status] :active))
      db)))

(rh/definvariant active-subscriptions-have-payment
  [{:keys [::subscriptions]}]
  (every? (fn [[_ sub]]
            (contains? #{:trial :active :cancelled} (:status sub)))
          subscriptions))
```

### Library Spec Design Principles

For Recife library models:

- keep input/output event shapes explicit
- avoid application-specific entities in library namespaces
- expose normalized events, not provider wire formats
- keep provider-specific parsing isolated
- include invariants that guarantee normalized event validity

## Using These Patterns

### Composition

Combine pattern components in one run:

```clojure
(comment
  @(r/run-model global
                #{password-auth/request-password-reset
                  rbac/authorize
                  notifications/batch-digest
                  usage-limits/create-project
                  comments/create-comment
                  password-auth/reset-token-user-exists
                  rbac/role-exists-for-user}))
```

Keep pattern state keys namespaced to avoid collisions.

### Adaptation

Adapt by changing:

- state keys and event shapes to your ubiquitous language
- config defaults (`::config` maps)
- guards/transition rules to domain policy
- invariants/properties to your explicit guarantees

### Anti-Patterns

Avoid:

- copying provider-specific API details into application model code
- mixing multiple queue/event formats for the same concern
- unbounded collections without invariants
- implicit status transitions not captured by process code
- reusing a pattern without renaming keys/terms to domain language
