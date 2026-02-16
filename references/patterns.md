# Complete patterns

This library contains reusable Recife model patterns for common SaaS scenarios. Each pattern mirrors the original behavioral intent while expressing it as executable Clojure model code.

Patterns elide common cross-cutting entities (`Email`, `Notification`, `AuditLog`, etc.) for brevity. In a real model, define these in shared namespaces.

| Pattern | Key Features Demonstrated |
|---------|---------------------------|
| Password Auth with Reset | Temporal triggers, token lifecycle, defaults, surfaces |
| Role-Based Access Control | Derived permissions, relationships, guard checks, surfaces |
| Invitation to Resource | Join entities, permission levels, invitation lifecycle, surfaces |
| Soft Delete & Restore | State machines, projections filtering deleted items |
| Notification Preferences | Notification variants, user preferences, digest batching, surfaces |
| Usage Limits & Quotas | Limit checks, metered resources, plan tiers, surfaces |
| Comments with Mentions | Nested entities, mention parsing, cross-entity notifications, surfaces |
| Integrating Library Specs | External model references, configuration, external trigger handling |

## Pattern 1: Password Authentication with Reset

File: `password-auth.model.clj`

```clojure
(ns patterns.password-auth.model
  (:require [recife.core :as r]))

(def global
  {::config {:min-password-length 12
             :max-login-attempts 5
             :lockout-duration-ms (* 15 60 1000)
             :reset-token-expiry-ms (* 60 60 1000)
             :session-duration-ms (* 24 60 60 1000)}
   ::users {}
   ::sessions {}
   ::reset-tokens {}
   :auth/register #{}
   :auth/login #{}
   :auth/logout #{}
   :auth/request-reset #{}
   :auth/complete-reset #{}
   :auth/notifications []
   :mail/outbox []
   :clock/now 0})

(defn user-by-email [db email]
  (first (filter (fn [[_ u]] (= email (:email u))) (::users db))))

(defn password-valid? [user password]
  (= (str "hash:" password) (:password-hash user)))

(defn user-locked? [db user]
  (and (= :locked (:status user))
       (some? (:locked-until user))
       (> (:locked-until user) (:clock/now db))))

(defn next-id [prefix m]
  (keyword (str prefix "-" (inc (count m)))))

(defn token-valid? [db token]
  (and (= :pending (:status token))
       (> (:expires-at token) (:clock/now db))))

(r/defproc register
  (fn [{:keys [:auth/register] :as db}]
    (when-let [{:keys [email password] :as cmd} (first register)]
      (when (and (nil? (user-by-email db email))
                 (>= (count password) (get-in db [::config :min-password-length])))
        (let [user-id (next-id "user" (::users db))
              now (:clock/now db)]
          (-> db
              (update :auth/register disj cmd)
              (assoc-in [::users user-id]
                        {:email email
                         :password-hash (str "hash:" password)
                         :status :active
                         :failed-login-attempts 0
                         :locked-until nil
                         :created-at now})
              (update :mail/outbox conj {:to email :template :welcome})))))))

(r/defproc login-success
  (fn [{:keys [:auth/login] :as db}]
    (when-let [{:keys [email password] :as cmd} (first login)]
      (when-let [[user-id user] (user-by-email db email)]
        (when (and (not (user-locked? db user))
                   (password-valid? user password))
          (let [now (:clock/now db)
                session-id (next-id "session" (::sessions db))]
            (-> db
                (update :auth/login disj cmd)
                (assoc-in [::users user-id :failed-login-attempts] 0)
                (assoc-in [::users user-id :status] :active)
                (assoc-in [::users user-id :locked-until] nil)
                (assoc-in [::sessions session-id]
                          {:user-id user-id
                           :created-at now
                           :expires-at (+ now (get-in db [::config :session-duration-ms]))
                           :status :active}))))))))

(r/defproc login-failure
  (fn [{:keys [:auth/login] :as db}]
    (when-let [{:keys [email password] :as cmd} (first login)]
      (when-let [[user-id user] (user-by-email db email)]
        (when (and (not (user-locked? db user))
                   (not (password-valid? user password)))
          (let [attempts (inc (:failed-login-attempts user))
                max-attempts (get-in db [::config :max-login-attempts])
                now (:clock/now db)
                db' (-> db
                        (update :auth/login disj cmd)
                        (assoc-in [::users user-id :failed-login-attempts] attempts))]
            (if (>= attempts max-attempts)
              (-> db'
                  (assoc-in [::users user-id :status] :locked)
                  (assoc-in [::users user-id :locked-until]
                            (+ now (get-in db [::config :lockout-duration-ms])))
                  (update :mail/outbox conj {:to (:email user)
                                             :template :account-locked}))
              db')))))))

(r/defproc login-attempt-while-locked
  (fn [{:keys [:auth/login] :as db}]
    (when-let [{:keys [email] :as cmd} (first login)]
      (when-let [[_ user] (user-by-email db email)]
        (when (user-locked? db user)
          (-> db
              (update :auth/login disj cmd)
              (update :auth/notifications conj
                      {:type :user-informed
                       :about :account-locked
                       :email email
                       :unlocks-at (:locked-until user)})))))))

(r/defproc lockout-expires
  (fn [{:keys [::users :clock/now] :as db}]
    (reduce-kv
     (fn [acc user-id user]
       (if (and (= :locked (:status user))
                (some? (:locked-until user))
                (<= (:locked-until user) now))
         (-> acc
             (assoc-in [::users user-id :status] :active)
             (assoc-in [::users user-id :failed-login-attempts] 0)
             (assoc-in [::users user-id :locked-until] nil))
         acc))
     db
     users)))

(r/defproc logout
  (fn [{:keys [:auth/logout] :as db}]
    (when-let [{:keys [session-id] :as cmd} (first logout)]
      (when (= :active (get-in db [::sessions session-id :status]))
        (-> db
            (update :auth/logout disj cmd)
            (assoc-in [::sessions session-id :status] :revoked))))))

(r/defproc session-expires
  (fn [{:keys [::sessions :clock/now] :as db}]
    (reduce-kv
     (fn [acc session-id session]
       (if (and (= :active (:status session))
                (<= (:expires-at session) now))
         (assoc-in acc [::sessions session-id :status] :expired)
         acc))
     db
     sessions)))

(r/defproc request-password-reset
  (fn [{:keys [:auth/request-reset] :as db}]
    (when-let [{:keys [email] :as cmd} (first request-reset)]
      (when-let [[user-id user] (user-by-email db email)]
        (when (contains? #{:active :locked} (:status user))
          (let [token-id (next-id "token" (::reset-tokens db))
                now (:clock/now db)
                db' (-> db
                        (update :auth/request-reset disj cmd)
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
                                   :expires-at (+ now (get-in db [::config :reset-token-expiry-ms]))
                                   :status :pending}))]
            (update db' :mail/outbox conj
                    {:to email :template :password-reset :data {:token-id token-id}})))))))

(r/defproc complete-password-reset
  (fn [{:keys [:auth/complete-reset] :as db}]
    (when-let [{:keys [token-id new-password] :as cmd} (first complete-reset)]
      (when-let [token (get-in db [::reset-tokens token-id])]
        (when (and (token-valid? db token)
                   (>= (count new-password) (get-in db [::config :min-password-length])))
          (let [user-id (:user-id token)
                user (get-in db [::users user-id])
                db' (-> db
                        (update :auth/complete-reset disj cmd)
                        (assoc-in [::reset-tokens token-id :status] :used)
                        (assoc-in [::users user-id :password-hash] (str "hash:" new-password))
                        (assoc-in [::users user-id :status] :active)
                        (assoc-in [::users user-id :failed-login-attempts] 0)
                        (assoc-in [::users user-id :locked-until] nil)
                        (update ::sessions
                                (fn [sessions]
                                  (into {}
                                        (map (fn [[sid session]]
                                               [sid (if (and (= user-id (:user-id session))
                                                             (= :active (:status session)))
                                                      (assoc session :status :revoked)
                                                      session)]))
                                        sessions))))]
            (update db' :mail/outbox conj
                    {:to (:email user) :template :password-changed})))))))

(r/defproc reset-token-expires
  (fn [{:keys [::reset-tokens :clock/now] :as db}]
    (reduce-kv
     (fn [acc token-id token]
       (if (and (= :pending (:status token))
                (<= (:expires-at token) now))
         (assoc-in acc [::reset-tokens token-id :status] :expired)
         acc))
     db
     reset-tokens)))
```

## Pattern 2: Role-Based Access Control (RBAC)

File: `rbac.model.clj`

```clojure
(ns patterns.rbac.model
  (:require [clojure.set :as set]
            [recife.core :as r]))

(def global
  {::roles {:viewer {:name "viewer"
                     :permissions #{"documents.read"}
                     :inherits-from nil}
            :editor {:name "editor"
                     :permissions #{"documents.write"}
                     :inherits-from :viewer}
            :admin {:name "admin"
                    :permissions #{"workspace.admin" "members.manage"}
                    :inherits-from :editor}}
   ::workspaces {}
   ::users {}
   ::workspace-memberships {}
   ::documents {}
   ::document-views {}
   :rbac/events #{}
   :mail/outbox []
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn effective-permissions [db role-id]
  (let [role (get-in db [::roles role-id])]
    (if-let [parent (:inherits-from role)]
      (set/union (:permissions role)
                 (effective-permissions db parent))
      (:permissions role))))

(defn membership [db workspace-id user-id]
  (get-in db [::workspace-memberships [workspace-id user-id]]))

(defn can-read? [db workspace-id user-id]
  (contains? (effective-permissions db (:role (membership db workspace-id user-id)))
             "documents.read"))

(defn can-write? [db workspace-id user-id]
  (contains? (effective-permissions db (:role (membership db workspace-id user-id)))
             "documents.write"))

(defn can-admin? [db workspace-id user-id]
  (contains? (effective-permissions db (:role (membership db workspace-id user-id)))
             "workspace.admin"))

(r/defproc create-workspace
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id name] :as cmd} (take-event events :create-workspace)]
      (let [workspace-id (keyword (str "workspace-" (inc (count (::workspaces db)))))
            now (:clock/now db)]
        (-> db
            (update :rbac/events disj cmd)
            (assoc-in [::workspaces workspace-id]
                      {:name name :owner-id user-id})
            (assoc-in [::workspace-memberships [workspace-id user-id]]
                      {:workspace-id workspace-id
                       :user-id user-id
                       :role :admin
                       :joined-at now}))))))

(r/defproc add-member
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [actor-id workspace-id new-user-id role] :as cmd}
               (take-event events :add-member)]
      (when (and (can-admin? db workspace-id actor-id)
                 (nil? (membership db workspace-id new-user-id)))
        (-> db
            (update :rbac/events disj cmd)
            (assoc-in [::workspace-memberships [workspace-id new-user-id]]
                      {:workspace-id workspace-id
                       :user-id new-user-id
                       :role role
                       :joined-at (:clock/now db)})
            (update :mail/outbox conj
                    {:to (get-in db [::users new-user-id :email])
                     :template :added-to-workspace
                     :data {:workspace-id workspace-id :role role}}))))))

(r/defproc change-member-role
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [actor-id workspace-id target-user-id new-role] :as cmd}
               (take-event events :change-member-role)]
      (when (and (can-admin? db workspace-id actor-id)
                 (some? (membership db workspace-id target-user-id))
                 (not= target-user-id (get-in db [::workspaces workspace-id :owner-id])))
        (-> db
            (update :rbac/events disj cmd)
            (assoc-in [::workspace-memberships [workspace-id target-user-id] :role] new-role))))))

(r/defproc remove-member
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [actor-id workspace-id target-user-id] :as cmd}
               (take-event events :remove-member)]
      (when (and (can-admin? db workspace-id actor-id)
                 (some? (membership db workspace-id target-user-id))
                 (not= target-user-id (get-in db [::workspaces workspace-id :owner-id])))
        (-> db
            (update :rbac/events disj cmd)
            (update ::workspace-memberships dissoc [workspace-id target-user-id]))))))

(r/defproc leave-workspace
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id workspace-id] :as cmd} (take-event events :leave-workspace)]
      (when (and (some? (membership db workspace-id user-id))
                 (not= user-id (get-in db [::workspaces workspace-id :owner-id])))
        (-> db
            (update :rbac/events disj cmd)
            (update ::workspace-memberships dissoc [workspace-id user-id]))))))

(r/defproc grant-permission
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [actor-id workspace-id role-id permission] :as cmd}
               (take-event events :grant-permission)]
      (when (and (can-admin? db workspace-id actor-id)
                 (not (contains? (effective-permissions db role-id) permission)))
        (-> db
            (update :rbac/events disj cmd)
            (update-in [::roles role-id :permissions] conj permission))))))

(r/defproc revoke-permission
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [actor-id workspace-id role-id permission] :as cmd}
               (take-event events :revoke-permission)]
      (when (and (can-admin? db workspace-id actor-id)
                 (contains? (get-in db [::roles role-id :permissions]) permission))
        (-> db
            (update :rbac/events disj cmd)
            (update-in [::roles role-id :permissions] disj permission))))))

(r/defproc create-document
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id workspace-id title content] :as cmd}
               (take-event events :create-document)]
      (when (can-write? db workspace-id user-id)
        (let [document-id (keyword (str "document-" (inc (count (::documents db)))))]
          (-> db
              (update :rbac/events disj cmd)
              (assoc-in [::documents document-id]
                        {:workspace-id workspace-id
                         :created-by user-id
                         :title title
                         :content content})))))))

(r/defproc view-document
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id document-id] :as cmd}
               (take-event events :view-document)]
      (let [workspace-id (get-in db [::documents document-id :workspace-id])]
        (when (can-read? db workspace-id user-id)
          (let [view-id (keyword (str "view-" (inc (count (::document-views db)))))]
            (-> db
                (update :rbac/events disj cmd)
                (assoc-in [::document-views view-id]
                          {:user-id user-id
                           :document-id document-id
                           :at (:clock/now db)}))))))))
```

## Pattern 3: Invitation to Resource

File: `resource-invitation.model.clj`

```clojure
(ns patterns.resource-invitation.model
  (:require [recife.core :as r]))

(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)}
   ::users {}
   ::resources {}
   ::resource-shares {}
   ::resource-invitations {}
   :resource/events #{}
   :mail/outbox []
   :notifications/outbox []
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn can-admin-resource? [db resource-id user-id]
  (or (= user-id (get-in db [::resources resource-id :owner-id]))
      (= :admin (get-in db [::resource-shares [resource-id user-id] :permission]))))

(defn can-invite? [db resource-id user-id]
  (or (= user-id (get-in db [::resources resource-id :owner-id]))
      (contains? #{:edit :admin}
                 (get-in db [::resource-shares [resource-id user-id] :permission]))))

(defn invitation-valid? [db invitation]
  (and (= :pending (:status invitation))
       (> (:expires-at invitation) (:clock/now db))))

(r/defproc invite-to-resource
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [inviter-id resource-id email permission] :as cmd}
               (take-event events :invite-to-resource)]
      (let [owner? (= inviter-id (get-in db [::resources resource-id :owner-id]))
            existing-active? (some (fn [[[rid uid] share]]
                                     (and (= resource-id rid)
                                          (= :active (:status share))
                                          (= email (get-in db [::users uid :email]))))
                                   (::resource-shares db))]
        (when (and (can-invite? db resource-id inviter-id)
                   (or (contains? #{:view :edit} permission)
                       (and (= :admin permission) owner?))
                   (not existing-active?))
          (let [inv-id (keyword (str "invitation-" (inc (count (::resource-invitations db)))))
                now (:clock/now db)]
            (-> db
                (update :resource/events disj cmd)
                (assoc-in [::resource-invitations inv-id]
                          {:resource-id resource-id
                           :email email
                           :permission permission
                           :invited-by inviter-id
                           :created-at now
                           :expires-at (+ now (get-in db [::config :invitation-expiry-ms]))
                           :status :pending})
                (update :mail/outbox conj
                        {:to email
                         :template :resource-invitation
                         :data {:resource-id resource-id
                                :invited-by inviter-id
                                :permission permission}}))))))))

(r/defproc accept-invitation-existing-user
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [invitation-id user-id] :as cmd}
               (take-event events :accept-invitation-existing-user)]
      (when-let [invitation (get-in db [::resource-invitations invitation-id])]
        (when (and (invitation-valid? db invitation)
                   (= (:email invitation) (get-in db [::users user-id :email])))
          (-> db
              (update :resource/events disj cmd)
              (assoc-in [::resource-invitations invitation-id :status] :accepted)
              (assoc-in [::resource-shares [(:resource-id invitation) user-id]]
                        {:resource-id (:resource-id invitation)
                         :user-id user-id
                         :permission (:permission invitation)
                         :status :active
                         :created-at (:clock/now db)})
              (update :notifications/outbox conj
                      {:to (:invited-by invitation)
                       :template :invitation-accepted
                       :data {:resource-id (:resource-id invitation)
                              :user-id user-id}})))))))

(r/defproc accept-invitation-new-user
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [invitation-id email name password] :as cmd}
               (take-event events :accept-invitation-new-user)]
      (when-let [invitation (get-in db [::resource-invitations invitation-id])]
        (let [existing-user? (some (fn [[_ user]] (= email (:email user))) (::users db))]
          (when (and (invitation-valid? db invitation)
                     (= email (:email invitation))
                     (not existing-user?))
            (let [user-id (keyword (str "user-" (inc (count (::users db)))))
                  now (:clock/now db)]
              (-> db
                  (update :resource/events disj cmd)
                  (assoc-in [::users user-id]
                            {:email email
                             :name name
                             :password-hash (str "hash:" password)
                             :status :active
                             :created-at now})
                  (assoc-in [::resource-invitations invitation-id :status] :accepted)
                  (assoc-in [::resource-shares [(:resource-id invitation) user-id]]
                            {:resource-id (:resource-id invitation)
                             :user-id user-id
                             :permission (:permission invitation)
                             :status :active
                             :created-at now})
                  (update :notifications/outbox conj
                          {:to (:invited-by invitation)
                           :template :invitation-accepted
                           :data {:resource-id (:resource-id invitation)
                                  :user-id user-id}})))))))))

(r/defproc decline-invitation
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [invitation-id] :as cmd}
               (take-event events :decline-invitation)]
      (when-let [invitation (get-in db [::resource-invitations invitation-id])]
        (when (invitation-valid? db invitation)
          (-> db
              (update :resource/events disj cmd)
              (assoc-in [::resource-invitations invitation-id :status] :declined)))))))

(r/defproc invitation-expires
  (fn [{:keys [::resource-invitations :clock/now] :as db}]
    (reduce-kv
     (fn [acc invitation-id invitation]
       (if (and (= :pending (:status invitation))
                (<= (:expires-at invitation) now))
         (assoc-in acc [::resource-invitations invitation-id :status] :expired)
         acc))
     db
     resource-invitations)))

(r/defproc revoke-invitation
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [actor-id invitation-id] :as cmd}
               (take-event events :revoke-invitation)]
      (when-let [invitation (get-in db [::resource-invitations invitation-id])]
        (when (and (= :pending (:status invitation))
                   (can-admin-resource? db (:resource-id invitation) actor-id))
          (-> db
              (update :resource/events disj cmd)
              (assoc-in [::resource-invitations invitation-id :status] :revoked)))))))

(r/defproc change-share-permission
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [actor-id resource-id target-user-id new-permission] :as cmd}
               (take-event events :change-share-permission)]
      (let [share-path [::resource-shares [resource-id target-user-id]]
            owner-id (get-in db [::resources resource-id :owner-id])]
        (when (and (can-admin-resource? db resource-id actor-id)
                   (some? (get-in db share-path))
                   (not= target-user-id owner-id)
                   (= :active (get-in db (conj share-path :status))))
          (-> db
              (update :resource/events disj cmd)
              (assoc-in (conj share-path :permission) new-permission)))))))

(r/defproc revoke-share
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [actor-id resource-id target-user-id] :as cmd}
               (take-event events :revoke-share)]
      (let [share-path [::resource-shares [resource-id target-user-id]]
            owner-id (get-in db [::resources resource-id :owner-id])]
        (when (and (can-admin-resource? db resource-id actor-id)
                   (some? (get-in db share-path))
                   (= :active (get-in db (conj share-path :status)))
                   (not= target-user-id owner-id))
          (-> db
              (update :resource/events disj cmd)
              (assoc-in (conj share-path :status) :revoked)
              (update :notifications/outbox conj
                      {:to target-user-id
                       :template :access-revoked
                       :data {:resource-id resource-id}})))))))
```

## Pattern 4: Soft Delete & Restore

File: `soft-delete.model.clj`

```clojure
(ns patterns.soft-delete.model
  (:require [recife.core :as r]))

(def global
  {::config {:retention-period-ms (* 30 24 60 60 1000)}
   ::documents {}
   ::workspace-memberships {}
   :document/events #{}
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn can-admin? [db workspace-id user-id]
  (true? (get-in db [::workspace-memberships [workspace-id user-id] :can-admin])))

(defn can-restore? [db document]
  (and (= :deleted (:status document))
       (some? (:deleted-at document))
       (> (+ (:deleted-at document)
             (get-in db [::config :retention-period-ms]))
          (:clock/now db))))

(r/defproc delete-document
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd}
               (take-event events :delete-document)]
      (when-let [document (get-in db [::documents document-id])]
        (when (and (= :active (:status document))
                   (or (= actor-id (:created-by document))
                       (can-admin? db (:workspace-id document) actor-id)))
          (-> db
              (update :document/events disj cmd)
              (assoc-in [::documents document-id :status] :deleted)
              (assoc-in [::documents document-id :deleted-at] (:clock/now db))
              (assoc-in [::documents document-id :deleted-by] actor-id)))))))

(r/defproc restore-document
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd}
               (take-event events :restore-document)]
      (when-let [document (get-in db [::documents document-id])]
        (when (and (can-restore? db document)
                   (or (= actor-id (:deleted-by document))
                       (can-admin? db (:workspace-id document) actor-id)))
          (-> db
              (update :document/events disj cmd)
              (assoc-in [::documents document-id :status] :active)
              (assoc-in [::documents document-id :deleted-at] nil)
              (assoc-in [::documents document-id :deleted-by] nil)))))))

(r/defproc permanently-delete
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd}
               (take-event events :permanently-delete)]
      (when-let [document (get-in db [::documents document-id])]
        (when (and (= :deleted (:status document))
                   (can-admin? db (:workspace-id document) actor-id))
          (-> db
              (update :document/events disj cmd)
              (update ::documents dissoc document-id)))))))

(r/defproc retention-expires
  (fn [{:keys [::documents :clock/now] :as db}]
    (reduce-kv
     (fn [acc document-id document]
       (if (and (= :deleted (:status document))
                (some? (:deleted-at document))
                (<= (+ (:deleted-at document)
                       (get-in db [::config :retention-period-ms]))
                    now))
         (update acc ::documents dissoc document-id)
         acc))
     db
     documents)))

(r/defproc empty-trash
  (fn [{:keys [:document/events ::documents] :as db}]
    (when-let [{:keys [actor-id workspace-id] :as cmd}
               (take-event events :empty-trash)]
      (when (can-admin? db workspace-id actor-id)
        (-> db
            (update :document/events disj cmd)
            (assoc ::documents
                   (into {}
                         (remove (fn [[_ document]]
                                   (and (= workspace-id (:workspace-id document))
                                        (= :deleted (:status document)))))
                         documents)))))))

(r/defproc restore-all
  (fn [{:keys [:document/events ::documents] :as db}]
    (when-let [{:keys [actor-id workspace-id] :as cmd}
               (take-event events :restore-all-deleted)]
      (when (can-admin? db workspace-id actor-id)
        (-> db
            (update :document/events disj cmd)
            (assoc ::documents
                   (reduce-kv
                    (fn [acc document-id document]
                      (if (and (= workspace-id (:workspace-id document))
                               (can-restore? db document))
                        (assoc acc document-id
                               (-> document
                                   (assoc :status :active)
                                   (assoc :deleted-at nil)
                                   (assoc :deleted-by nil)))
                        (assoc acc document-id document)))
                    {}
                    documents)))))))
```

## Pattern 5: Notification Preferences & Digests

File: `notifications.model.clj`

```clojure
(ns patterns.notifications.model
  (:require [recife.core :as r]))

(def global
  {::users {}
   ::notification-settings {}
   ::notifications {}
   ::digest-batches {}
   :notification/events #{}
   :mail/outbox []
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn next-id [prefix m]
  (keyword (str prefix "-" (inc (count m)))))

(defn preference-for [settings kind]
  (case kind
    :mention (:email-on-mention settings)
    :reply (:email-on-comment settings)
    :share (:email-on-share settings)
    :assignment (:email-on-assignment settings)
    :immediately))

(defn create-notification [db user-id kind attrs]
  (let [notification-id (next-id "notification" (::notifications db))
        settings (get-in db [::notification-settings user-id])
        preference (preference-for settings kind)
        email-status (if (= :never preference) :skipped :pending)]
    (assoc-in db [::notifications notification-id]
              (merge {:id notification-id
                      :user-id user-id
                      :kind kind
                      :created-at (:clock/now db)
                      :status :unread
                      :email-status email-status}
                     attrs))))

(r/defproc create-mention-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id comment-id mentioned-by] :as cmd}
               (take-event events :user-mentioned)]
      (when (not= user-id mentioned-by)
        (-> db
            (update :notification/events disj cmd)
            (create-notification user-id :mention
                                 {:comment-id comment-id
                                  :mentioned-by mentioned-by}))))))

(r/defproc create-reply-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [original-author-id reply-id original-comment-id replied-by] :as cmd}
               (take-event events :comment-replied)]
      (when (not= original-author-id replied-by)
        (-> db
            (update :notification/events disj cmd)
            (create-notification original-author-id :reply
                                 {:reply-id reply-id
                                  :original-comment-id original-comment-id
                                  :replied-by replied-by}))))))

(r/defproc create-share-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id resource-id shared-by permission] :as cmd}
               (take-event events :resource-shared)]
      (when (not= user-id shared-by)
        (-> db
            (update :notification/events disj cmd)
            (create-notification user-id :share
                                 {:resource-id resource-id
                                  :shared-by shared-by
                                  :permission permission}))))))

(r/defproc create-assignment-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id task-id assigned-by] :as cmd}
               (take-event events :task-assigned)]
      (when (not= user-id assigned-by)
        (-> db
            (update :notification/events disj cmd)
            (create-notification user-id :assignment
                                 {:task-id task-id
                                  :assigned-by assigned-by}))))))

(r/defproc create-system-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id title body link] :as cmd}
               (take-event events :system-notification-triggered)]
      (-> db
          (update :notification/events disj cmd)
          (create-notification user-id :system
                               {:title title :body body :link link
                                :email-status :pending})))))

(r/defproc send-immediate-email
  (fn [{:keys [::notifications] :as db}]
    (when-let [[notification-id notification]
               (first (filter
                       (fn [[_ notification]]
                         (let [settings (get-in db [::notification-settings (:user-id notification)])
                               preference (preference-for settings (:kind notification))]
                           (and (= :pending (:email-status notification))
                                (= :immediately preference))))
                       notifications))]
      (-> db
          (assoc-in [::notifications notification-id :email-status] :sent)
          (update :mail/outbox conj
                  {:to (get-in db [::users (:user-id notification) :email])
                   :template :notification-immediate
                   :data {:notification-id notification-id}})))))

(r/defproc mark-as-read
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id notification-id] :as cmd}
               (take-event events :mark-notification-read)]
      (when (and (= user-id (get-in db [::notifications notification-id :user-id]))
                 (= :unread (get-in db [::notifications notification-id :status])))
        (-> db
            (update :notification/events disj cmd)
            (assoc-in [::notifications notification-id :status] :read))))))

(r/defproc mark-all-as-read
  (fn [{:keys [:notification/events ::notifications] :as db}]
    (when-let [{:keys [user-id] :as cmd} (take-event events :mark-all-notifications-read)]
      (-> db
          (update :notification/events disj cmd)
          (assoc ::notifications
                 (reduce-kv
                  (fn [acc notification-id notification]
                    (if (and (= user-id (:user-id notification))
                             (= :unread (:status notification)))
                      (assoc acc notification-id (assoc notification :status :read))
                      (assoc acc notification-id notification)))
                  {}
                  notifications))))))

(r/defproc archive-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id notification-id] :as cmd}
               (take-event events :archive-notification)]
      (when (= user-id (get-in db [::notifications notification-id :user-id]))
        (-> db
            (update :notification/events disj cmd)
            (assoc-in [::notifications notification-id :status] :archived))))))

(r/defproc create-daily-digest
  (fn [{:keys [::users ::notifications :clock/now] :as db}]
    (reduce-kv
     (fn [acc user-id user]
       (let [settings (get-in acc [::notification-settings user-id])
             pending (filter (fn [[_ notification]]
                               (and (= user-id (:user-id notification))
                                    (= :pending (:email-status notification))
                                    (>= (:created-at notification)
                                        (- now (* 24 60 60 1000)))))
                             (::notifications acc))]
         (if (and (:digest-enabled settings)
                  (some? (:next-digest-at user))
                  (<= (:next-digest-at user) now)
                  (seq pending))
           (let [batch-id (next-id "digest" (::digest-batches acc))]
             (let [acc' (-> acc
                            (assoc-in [::digest-batches batch-id]
                                      {:user-id user-id
                                       :notification-ids (mapv first pending)
                                       :created-at now
                                       :sent-at nil
                                       :status :pending})
                            (assoc-in [::users user-id :next-digest-at]
                                      (+ now (* 24 60 60 1000))))]
               (reduce (fn [x [notification-id _]]
                         (assoc-in x [::notifications notification-id :email-status] :digested))
                       acc'
                       pending)))
           acc)))
     db
     users)))

(r/defproc send-digest
  (fn [{:keys [::digest-batches] :as db}]
    (when-let [[batch-id batch]
               (first (filter (fn [[_ batch]]
                                (and (= :pending (:status batch))
                                     (seq (:notification-ids batch))))
                              digest-batches))]
      (-> db
          (assoc-in [::digest-batches batch-id :status] :sent)
          (assoc-in [::digest-batches batch-id :sent-at] (:clock/now db))
          (update :mail/outbox conj
                  {:to (get-in db [::users (:user-id batch) :email])
                   :template :daily-digest
                   :data {:notification-ids (:notification-ids batch)
                          :unread-count (count (filter (fn [[_ notification]]
                                                         (and (= (:user-id batch) (:user-id notification))
                                                              (= :unread (:status notification))))
                                                       (::notifications db)))}})))))

(r/defproc update-notification-preferences
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id preferences] :as cmd}
               (take-event events :update-preferences)]
      (-> db
          (update :notification/events disj cmd)
          (assoc-in [::notification-settings user-id :email-on-mention] (:mention preferences))
          (assoc-in [::notification-settings user-id :email-on-comment] (:comment preferences))
          (assoc-in [::notification-settings user-id :email-on-share] (:share preferences))
          (assoc-in [::notification-settings user-id :email-on-assignment] (:assignment preferences))
          (assoc-in [::notification-settings user-id :digest-enabled] (:digest-enabled preferences))
          (assoc-in [::notification-settings user-id :digest-day-of-week] (:digest-days preferences))))))
```

## Pattern 6: Usage Limits & Quotas

File: `usage-limits.model.clj`

```clojure
(ns patterns.usage-limits.model
  (:require [recife.core :as r]))

(def global
  {::plans {}
   ::workspaces {}
   ::workspace-usage {}
   ::workspace-memberships {}
   ::documents {}
   ::usage-events []
   :usage/events #{}
   :notifications/outbox []
   :mail/outbox []
   :api/responses []
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn unlimited? [v]
  (nil? v))

(defn workspace-doc-count [db workspace-id]
  (->> (::documents db)
       vals
       (filter #(= workspace-id (:workspace-id %)))
       count))

(defn workspace-member-count [db workspace-id]
  (->> (::workspace-memberships db)
       keys
       (filter #(= workspace-id (first %)))
       count))

(defn workspace-plan [db workspace-id]
  (get-in db [::plans (get-in db [::workspaces workspace-id :plan-id])]))

(defn can-add-document? [db workspace-id]
  (let [max-documents (:max-documents (workspace-plan db workspace-id))]
    (or (unlimited? max-documents)
        (< (workspace-doc-count db workspace-id) max-documents))))

(defn can-add-member? [db workspace-id]
  (let [max-members (:max-team-members (workspace-plan db workspace-id))]
    (or (unlimited? max-members)
        (< (workspace-member-count db workspace-id) max-members))))

(defn can-use-feature? [db workspace-id feature]
  (contains? (:features (workspace-plan db workspace-id)) feature))

(defn over-api-quota? [db workspace-id]
  (let [max-requests (:max-api-requests-per-day (workspace-plan db workspace-id))
        used (get-in db [::workspace-usage workspace-id :api-requests-today] 0)]
    (and (some? max-requests)
         (>= used max-requests))))

(r/defproc create-document
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id title] :as cmd}
               (take-event events :create-document)]
      (when (can-add-document? db workspace-id)
        (let [document-id (keyword (str "document-" (inc (count (::documents db)))))
              now (:clock/now db)]
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::documents document-id]
                        {:workspace-id workspace-id
                         :created-by user-id
                         :title title
                         :created-at now})
              (update ::usage-events conj
                      {:workspace-id workspace-id
                       :type :document-created
                       :amount 1
                       :recorded-at now})))))))

(r/defproc create-document-limit-reached
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id] :as cmd}
               (take-event events :create-document)]
      (when (not (can-add-document? db workspace-id))
        (let [plan (workspace-plan db workspace-id)]
          (-> db
              (update :usage/events disj cmd)
              (update :notifications/outbox conj
                      {:user-id user-id
                       :about :limit-reached
                       :data {:limit-type :documents
                              :current (workspace-doc-count db workspace-id)
                              :max (:max-documents plan)}})))))))

(r/defproc add-team-member
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [actor-id workspace-id new-member-id role] :as cmd}
               (take-event events :add-member)]
      (when (and (can-add-member? db workspace-id)
                 (true? (get-in db [::workspace-memberships [workspace-id actor-id] :can-admin])))
        (let [now (:clock/now db)]
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::workspace-memberships [workspace-id new-member-id]]
                        {:workspace-id workspace-id
                         :user-id new-member-id
                         :role role
                         :can-admin (= :admin role)
                         :joined-at now})
              (update ::usage-events conj
                      {:workspace-id workspace-id
                       :type :member-added
                       :amount 1
                       :recorded-at now})))))))

(r/defproc use-feature
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id feature] :as cmd}
               (take-event events :use-feature)]
      (when (can-use-feature? db workspace-id feature)
        (-> db
            (update :usage/events disj cmd)
            (update :notifications/outbox conj
                    {:user-id user-id
                     :about :feature-used
                     :data {:workspace-id workspace-id
                            :feature feature}}))))))

(r/defproc use-feature-not-available
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id feature] :as cmd}
               (take-event events :use-feature)]
      (when (not (can-use-feature? db workspace-id feature))
        (-> db
            (update :usage/events disj cmd)
            (update :notifications/outbox conj
                    {:user-id user-id
                     :about :feature-not-available
                     :data {:workspace-id workspace-id
                            :feature feature}}))))))

(r/defproc record-api-request
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id endpoint] :as cmd}
               (take-event events :api-request-received)]
      (when (not (over-api-quota? db workspace-id))
        (let [now (:clock/now db)]
          (-> db
              (update :usage/events disj cmd)
              (update-in [::workspace-usage workspace-id :api-requests-today] (fnil inc 0))
              (update ::usage-events conj
                      {:workspace-id workspace-id
                       :type :api-request
                       :amount 1
                       :endpoint endpoint
                       :recorded-at now})))))))

(r/defproc api-rate-limit-exceeded
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id] :as cmd}
               (take-event events :api-request-received)]
      (when (over-api-quota? db workspace-id)
        (-> db
            (update :usage/events disj cmd)
            (update :api/responses conj
                    {:status 429
                     :body {:error "rate_limit_exceeded"
                            :resets-at (get-in db [::workspace-usage workspace-id :next-reset-at])}}))))))

(r/defproc reset-daily-api-usage
  (fn [{:keys [::workspace-usage :clock/now] :as db}]
    (reduce-kv
     (fn [acc workspace-id usage]
       (if (and (some? (:next-reset-at usage))
                (<= (:next-reset-at usage) now))
         (-> acc
             (assoc-in [::workspace-usage workspace-id :api-requests-today] 0)
             (assoc-in [::workspace-usage workspace-id :next-reset-at]
                       (+ (:next-reset-at usage) (* 24 60 60 1000))))
         acc))
     db
     workspace-usage)))

(r/defproc upgrade-plan
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id new-plan-id] :as cmd}
               (take-event events :upgrade-plan)]
      (let [old-plan (workspace-plan db workspace-id)
            new-plan (get-in db [::plans new-plan-id])
            old-limit (:max-documents old-plan)
            new-limit (:max-documents new-plan)
            non-decreasing? (or (unlimited? new-limit)
                                (and (some? old-limit) (<= old-limit new-limit)))]
        (when non-decreasing?
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::workspaces workspace-id :plan-id] new-plan-id)
              (update :mail/outbox conj
                      {:to (get-in db [::workspaces workspace-id :owner-email])
                       :template :plan-upgraded
                       :data {:old-plan old-plan :new-plan new-plan}})))))))

(r/defproc downgrade-plan
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id new-plan-id] :as cmd}
               (take-event events :downgrade-plan)]
      (let [new-plan (get-in db [::plans new-plan-id])
            documents-ok (or (unlimited? (:max-documents new-plan))
                             (<= (workspace-doc-count db workspace-id)
                                 (:max-documents new-plan)))
            members-ok (or (unlimited? (:max-team-members new-plan))
                           (<= (workspace-member-count db workspace-id)
                               (:max-team-members new-plan)))
            storage-used (get-in db [::workspace-usage workspace-id :storage-bytes-used] 0)
            storage-ok (or (unlimited? (:max-storage-bytes new-plan))
                           (<= storage-used (:max-storage-bytes new-plan)))]
        (when (and documents-ok members-ok storage-ok)
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::workspaces workspace-id :plan-id] new-plan-id)
              (update :mail/outbox conj
                      {:to (get-in db [::workspaces workspace-id :owner-email])
                       :template :plan-downgraded
                       :data {:new-plan new-plan}})))))))

(r/defproc downgrade-blocked
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id] :as cmd}
               (take-event events :downgrade-plan)]
      (let [new-plan (get-in db [::plans (:new-plan-id cmd)])
            over-documents (and (not (unlimited? (:max-documents new-plan)))
                                (> (workspace-doc-count db workspace-id)
                                   (:max-documents new-plan)))
            over-members (and (not (unlimited? (:max-team-members new-plan)))
                              (> (workspace-member-count db workspace-id)
                                 (:max-team-members new-plan)))
            over-storage (let [storage-used (get-in db [::workspace-usage workspace-id :storage-bytes-used] 0)]
                           (and (not (unlimited? (:max-storage-bytes new-plan)))
                                (> storage-used (:max-storage-bytes new-plan))))]
        (when (or over-documents over-members over-storage)
          (-> db
              (update :usage/events disj cmd)
              (update :notifications/outbox conj
                      {:user-id (get-in db [::workspaces workspace-id :owner-id])
                       :about :downgrade-blocked
                       :data {:over-documents over-documents
                              :over-members over-members
                              :over-storage over-storage}})))))))
```

## Pattern 7: Comments with Mentions

File: `comments.model.clj`

```clojure
(ns patterns.comments.model
  (:require [clojure.set :as set]
            [clojure.string :as str]
            [recife.core :as r]))

(def global
  {::users {}
   ::comments {}
   ::comment-mentions {}
   ::comment-reactions {}
   :comments/events #{}
   :notifications/events #{}
   :notifications/outbox []
   :clock/now 0})

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn parse-mentions [body]
  (->> (str/split body #"\\s+")
       (filter #(str/starts-with? % "@"))
       (map #(keyword (subs % 1)))
       set))

(defn thread-depth [db comment-id]
  (loop [depth 0 cid comment-id]
    (if-let [parent-id (get-in db [::comments cid :reply-to])]
      (recur (inc depth) parent-id)
      depth)))

(r/defproc create-comment
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [author-id parent-id body] :as cmd}
               (take-event events :create-comment)]
      (let [comment-id (keyword (str "comment-" (inc (count (::comments db)))))
            mentions (parse-mentions body)
            now (:clock/now db)]
        (let [db'' (-> db
                       (update :comments/events disj cmd)
                       (assoc-in [::comments comment-id]
                                 {:parent-id parent-id
                                  :reply-to nil
                                  :author-id author-id
                                  :body body
                                  :created-at now
                                  :edited-at nil
                                  :status :active}))
              db' (reduce (fn [acc user-id]
                            (assoc-in acc [::comment-mentions [comment-id user-id]]
                                      {:comment-id comment-id
                                       :user-id user-id
                                       :notified false}))
                          db''
                          mentions)]
          (reduce (fn [acc user-id]
                    (update acc :notifications/events conj
                            {:type :mention-created
                             :comment-id comment-id
                             :user-id user-id}))
                  db'
                  mentions))))))

(r/defproc create-reply
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [author-id parent-comment-id body] :as cmd}
               (take-event events :create-reply)]
      (when-let [parent-comment (get-in db [::comments parent-comment-id])]
        (when (and (= :active (:status parent-comment))
                   (< (thread-depth db parent-comment-id) 3))
          (let [comment-id (keyword (str "comment-" (inc (count (::comments db)))))
                mentions (parse-mentions body)
                now (:clock/now db)]
            (let [db'' (-> db
                           (update :comments/events disj cmd)
                           (assoc-in [::comments comment-id]
                                     {:parent-id (:parent-id parent-comment)
                                      :reply-to parent-comment-id
                                      :author-id author-id
                                      :body body
                                      :created-at now
                                      :edited-at nil
                                      :status :active})
                           (update :notifications/events conj
                                   {:type :reply-created
                                    :reply-id comment-id
                                    :parent-comment-id parent-comment-id}))
                  db' (reduce (fn [acc user-id]
                                (assoc-in acc [::comment-mentions [comment-id user-id]]
                                          {:comment-id comment-id
                                           :user-id user-id
                                           :notified false}))
                              db''
                              mentions)]
              (reduce (fn [acc user-id]
                        (update acc :notifications/events conj
                                {:type :mention-created
                                 :comment-id comment-id
                                 :user-id user-id}))
                      db'
                      mentions))))))))

(r/defproc notify-mentioned-user
  (fn [{:keys [:notifications/events] :as db}]
    (when-let [{:keys [comment-id user-id] :as cmd}
               (take-event events :mention-created)]
      (let [mention-path [::comment-mentions [comment-id user-id]]
            author-id (get-in db [::comments comment-id :author-id])]
        (when (and (not= user-id author-id)
                   (not (true? (get-in db (conj mention-path :notified)))))
          (-> db
              (update :notifications/events disj cmd)
              (assoc-in (conj mention-path :notified) true)
              (update :notifications/outbox conj
                      {:to user-id
                       :type :mention
                       :comment-id comment-id
                       :mentioned-by author-id})))))))

(r/defproc notify-comment-author-of-reply
  (fn [{:keys [:notifications/events] :as db}]
    (when-let [{:keys [reply-id parent-comment-id] :as cmd}
               (take-event events :reply-created)]
      (let [reply (get-in db [::comments reply-id])
            parent (get-in db [::comments parent-comment-id])
            original-author (:author-id parent)
            reply-author (:author-id reply)
            mentioned-users (->> (::comment-mentions db)
                                 (filter (fn [[[comment-id _] _]] (= comment-id reply-id)))
                                 (map (fn [[[ _ user-id] _]] user-id))
                                 set)]
        (when (and (some? original-author)
                   (not= original-author reply-author)
                   (not (contains? mentioned-users original-author)))
          (-> db
              (update :notifications/events disj cmd)
              (update :notifications/outbox conj
                      {:to original-author
                       :type :reply
                       :reply-id reply-id
                       :original-comment-id parent-comment-id})))))))

(r/defproc edit-comment
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [actor-id comment-id new-body] :as cmd}
               (take-event events :edit-comment)]
      (let [comment (get-in db [::comments comment-id])]
        (when (and (= actor-id (:author-id comment))
                   (= :active (:status comment)))
          (let [old-mentions (->> (::comment-mentions db)
                                  keys
                                  (filter (fn [[cid _]] (= cid comment-id)))
                                  (map second)
                                  set)
                new-mentions (parse-mentions new-body)
                removed-mentions (set/difference old-mentions new-mentions)
                added-mentions (set/difference new-mentions old-mentions)
                now (:clock/now db)]
            (let [db' (-> db
                          (update :comments/events disj cmd)
                          (assoc-in [::comments comment-id :body] new-body)
                          (assoc-in [::comments comment-id :edited-at] now)
                          (update ::comment-mentions
                                  (fn [mentions]
                                    (reduce (fn [acc user-id]
                                              (dissoc acc [comment-id user-id]))
                                            mentions
                                            removed-mentions))))]
              (reduce (fn [acc user-id]
                        (assoc-in acc [::comment-mentions [comment-id user-id]]
                                  {:comment-id comment-id
                                   :user-id user-id
                                   :notified false}))
                      db'
                      added-mentions))))))))

(r/defproc delete-comment
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [actor-id comment-id] :as cmd}
               (take-event events :delete-comment)]
      (let [comment (get-in db [::comments comment-id])
            actor-admin? (true? (get-in db [::users actor-id :is-admin]))]
        (when (and (= :active (:status comment))
                   (or (= actor-id (:author-id comment))
                       actor-admin?))
          (-> db
              (update :comments/events disj cmd)
              (assoc-in [::comments comment-id :status] :deleted)))))))

(r/defproc add-reaction
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [user-id comment-id emoji] :as cmd}
               (take-event events :add-reaction)]
      (when (and (= :active (get-in db [::comments comment-id :status]))
                 (nil? (get-in db [::comment-reactions [comment-id user-id emoji]])))
        (-> db
            (update :comments/events disj cmd)
            (assoc-in [::comment-reactions [comment-id user-id emoji]]
                      {:comment-id comment-id
                       :user-id user-id
                       :emoji emoji
                       :created-at (:clock/now db)}))))))

(r/defproc remove-reaction
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [user-id comment-id emoji] :as cmd}
               (take-event events :remove-reaction)]
      (when (and (= :active (get-in db [::comments comment-id :status]))
                 (some? (get-in db [::comment-reactions [comment-id user-id emoji]])))
        (-> db
            (update :comments/events disj cmd)
            (update ::comment-reactions dissoc [comment-id user-id emoji]))))))

(r/defproc toggle-reaction
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [user-id comment-id emoji] :as cmd}
               (take-event events :toggle-reaction)]
      (when (= :active (get-in db [::comments comment-id :status]))
        (let [reaction-path [::comment-reactions [comment-id user-id emoji]]]
          (-> db
              (update :comments/events disj cmd)
              ((fn [x]
                 (if (some? (get-in x reaction-path))
                   (update x ::comment-reactions dissoc [comment-id user-id emoji])
                   (assoc-in x reaction-path
                             {:comment-id comment-id
                              :user-id user-id
                              :emoji emoji
                              :created-at (:clock/now x)}))))))))))
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
            [recife.core :as r]))

(def global
  (merge oauth/global
         {::users {}
          ::user-preferences {}
          :app/events #{}
          :app/notifications []
          :audit/outbox []
          :mail/outbox []
          :clock/now 0}))

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn user-by-email [db email]
  (first (filter (fn [[_ user]] (= email (:email user))) (::users db))))

(r/defproc create-user-on-first-login
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (take-event events :authentication-succeeded)]
      (when-not (user-by-email db (:email event))
        (let [user-id (keyword (str "user-" (inc (count (::users db)))))
              now (:clock/now db)]
          (-> db
              (update :oauth/events disj event)
              (assoc-in [::users user-id]
                        {:email (:email event)
                         :name (:display-name event)
                         :avatar-url (:avatar-url event)
                         :status :active
                         :created-at now
                         :last-login-at now})
              (assoc-in [::user-preferences user-id]
                        {:theme :system
                         :timezone (or (:timezone event) "UTC")
                         :locale (or (:locale event) "en")})
              (assoc-in [:oauth/identities (:identity-id event) :user-id] user-id)
              (assoc-in [:oauth/sessions (:session-id event) :user-id] user-id)
              (update :mail/outbox conj
                      {:to (:email event)
                       :template :welcome
                       :data {:provider (:provider event)}})))))))

(r/defproc update-user-on-login
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (take-event events :authentication-succeeded)]
      (when-let [[user-id user] (user-by-email db (:email event))]
        (when (= :active (:status user))
          (-> db
              (update :oauth/events disj event)
              (assoc-in [::users user-id :last-login-at] (:clock/now db))
              (assoc-in [:oauth/sessions (:session-id event) :user-id] user-id)))))))

(r/defproc block-suspended-user-login
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (take-event events :authentication-succeeded)]
      (when-let [[user-id user] (user-by-email db (:email event))]
        (when (= :suspended (:status user))
          (-> db
              (update :oauth/events disj event)
              (assoc-in [:oauth/sessions (:session-id event) :status] :revoked)
              (update :app/notifications conj
                      {:type :user-informed
                       :user-id user-id
                       :about :account-suspended
                       :data {:contact "support@example.com"}})))))))

(r/defproc notify-session-expiring
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (take-event events :session-status-changed)]
      (when (= :expiring (:status event))
        (when-let [user-id (get-in db [:oauth/sessions (:session-id event) :user-id])]
          (-> db
              (update :oauth/events disj event)
              (update :app/notifications conj
                      {:type :user-informed
                       :user-id user-id
                       :about :session-expiring
                       :data {:time-remaining (:time-remaining event)}})))))))

(r/defproc audit-logout
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (take-event events :session-terminated)]
      (when-let [user-id (get-in db [:oauth/sessions (:session-id event) :user-id])]
        (-> db
            (update :oauth/events disj event)
            (update :audit/outbox conj
                    {:user-id user-id
                     :event :logout
                     :reason (:reason event)
                     :timestamp (:clock/now db)
                     :metadata {:provider (:provider event)
                                :session-start (:created-at event)}}))))))

(r/defproc link-additional-provider
  (fn [{:keys [:app/events] :as db}]
    (when-let [{:keys [user-id provider] :as cmd}
               (take-event events :link-provider)]
      (let [user (get-in db [::users user-id])
            linked-providers (->> (:oauth/identities db)
                                  vals
                                  (filter #(= user-id (:user-id %)))
                                  (map :provider)
                                  set)]
        (when (and (= :active (:status user))
                   (not (contains? linked-providers provider)))
          (-> db
              (update :app/events disj cmd)
              (update :oauth/events conj
                      {:type :initiate-authentication
                       :provider provider
                       :intent :link-account
                       :existing-user-id user-id})))))))

(r/defproc unlink-provider
  (fn [{:keys [:app/events] :as db}]
    (when-let [{:keys [user-id provider] :as cmd}
               (take-event events :unlink-provider)]
      (let [identities (filter (fn [[_ identity]]
                                 (= user-id (:user-id identity)))
                               (:oauth/identities db))
            identity-id (first (keep (fn [[identity-id identity]]
                                       (when (and (= user-id (:user-id identity))
                                                  (= provider (:provider identity)))
                                         identity-id))
                                     (:oauth/identities db)))]
        (when (and (some? identity-id)
                   (> (count identities) 1)
                   (= :active (get-in db [::users user-id :status])))
          (-> db
              (update :app/events disj cmd)
              (update :oauth/identities dissoc identity-id)
              (update :audit/outbox conj
                      {:user-id user-id
                       :event :provider-unlinked
                       :timestamp (:clock/now db)
                       :metadata {:provider provider}})))))))
```

### Example: Payment Processing

Files:

- `stripe-billing/model.clj` (library model)
- `billing/model.clj` (application model)

```clojure
(ns billing.model
  (:require [recife.core :as r]
            [stripe-billing.model :as stripe]))

(def global
  (merge stripe/global
         {::organisations {}
          ::subscriptions {}
          ::documents {}
          :billing/events #{}
          :mail/outbox []
          :audit/outbox []
          :app/notifications []
          :clock/now 0}))

(defn take-event [events event-type]
  (first (filter #(= event-type (:type %)) events)))

(defn org-id-by-customer [db customer-id]
  (first (keep (fn [[org-id org]]
                 (when (= customer-id (:stripe-customer-id org))
                   org-id))
               (::organisations db))))

(defn org-sub-id [db org-id]
  (first (keep (fn [[sub-id sub]]
                 (when (= org-id (:organisation-id sub))
                   sub-id))
               (::subscriptions db))))

(r/defproc activate-on-payment-success
  (fn [{:keys [:stripe/events] :as db}]
    (when-let [event (take-event events :payment-succeeded)]
      (let [org-id (org-id-by-customer db (:customer-id event))
            sub-id (some-> org-id (org-sub-id db))]
        (when (and org-id sub-id
                   (contains? #{:trialing :past-due}
                              (get-in db [::subscriptions sub-id :status])))
          (-> db
              (update :stripe/events disj event)
              (assoc-in [::subscriptions sub-id :status] :active)
              (assoc-in [::subscriptions sub-id :current-period-ends-at]
                        (:period-end event))
              (update :mail/outbox conj
                      {:to (get-in db [::organisations org-id :owner-email])
                       :template :payment-confirmed
                       :data {:amount (:amount event)
                              :next-billing (:period-end event)}})))))))

(r/defproc handle-payment-failure
  (fn [{:keys [:stripe/events] :as db}]
    (when-let [event (take-event events :payment-failed)]
      (let [org-id (org-id-by-customer db (:customer-id event))
            sub-id (some-> org-id (org-sub-id db))]
        (when (and org-id sub-id)
          (-> db
              (update :stripe/events disj event)
              (assoc-in [::subscriptions sub-id :status] :past-due)
              (update :mail/outbox conj
                      {:to (get-in db [::organisations org-id :owner-email])
                       :template :payment-failed
                       :data {:reason (:failure-reason event)
                              :retry-date (:next-payment-attempt event)
                              :update-payment-url (get-in db [::organisations org-id :billing-portal-url])}})
              (update :app/notifications conj
                      {:type :user-informed
                       :user-id (get-in db [::organisations org-id :owner-id])
                       :about :payment-failed
                       :data {:reason (:failure-reason event)}})))))))

(r/defproc trial-ending-reminder
  (fn [{:keys [::subscriptions :clock/now] :as db}]
    (reduce-kv
     (fn [acc sub-id sub]
       (if (and (= :trialing (:status sub))
                (not (:trial-reminder-sent sub))
                (some? (:trial-ends-at sub))
                (<= (- (:trial-ends-at sub) (* 3 24 60 60 1000)) now))
         (let [org-id (:organisation-id sub)]
           (-> acc
               (assoc-in [::subscriptions sub-id :trial-reminder-sent] true)
               (update :mail/outbox conj
                       {:to (get-in acc [::organisations org-id :owner-email])
                        :template :trial-ending
                        :data {:days-remaining 3
                               :plan (:plan-id sub)
                               :has-payment-method
                               (true? (get-in acc [::organisations org-id :has-payment-method]))}})))
         acc))
     db
     subscriptions)))

(r/defproc handle-subscription-cancelled
  (fn [{:keys [:stripe/events] :as db}]
    (when-let [event (take-event events :subscription-cancelled)]
      (let [sub-id (first (keep (fn [[sub-id sub]]
                                  (when (= (:stripe-subscription-id sub)
                                           (:stripe-subscription-id event))
                                    sub-id))
                                (::subscriptions db)))
            org-id (get-in db [::subscriptions sub-id :organisation-id])]
        (when (and sub-id org-id)
          (-> db
              (update :stripe/events disj event)
              (assoc-in [::subscriptions sub-id :status] :cancelled)
              (update :mail/outbox conj
                      {:to (get-in db [::organisations org-id :owner-email])
                       :template :subscription-cancelled
                       :data {:reason (:reason event)
                              :access-until (get-in db [::subscriptions sub-id :current-period-ends-at])}})
              (update :audit/outbox conj
                      {:user-id (get-in db [::organisations org-id :owner-id])
                       :event :subscription-cancelled
                       :timestamp (:clock/now db)
                       :metadata {:reason (:reason event)
                                  :plan (get-in db [::subscriptions sub-id :plan-id])}})))))))

(r/defproc start-subscription
  (fn [{:keys [:billing/events] :as db}]
    (when-let [{:keys [org-id plan-id] :as cmd}
               (take-event events :start-subscription)]
      (let [sub-id (org-sub-id db org-id)
            org (get-in db [::organisations org-id])
            sub (and sub-id (get-in db [::subscriptions sub-id]))]
        (when (and (or (nil? sub)
                       (contains? #{:cancelled :expired} (:status sub)))
                   (some? (:stripe-customer-id org))
                   (true? (:has-payment-method org)))
          (-> db
              (update :billing/events disj cmd)
              (update :stripe/events conj
                      {:type :create-subscription
                       :customer-id (:stripe-customer-id org)
                       :price-id (:stripe-price-id plan-id)
                       :trial-period-days (if (:has-trial plan-id) 14 nil)})))))))

(r/defproc change-plan
  (fn [{:keys [:billing/events] :as db}]
    (when-let [{:keys [org-id new-plan-id] :as cmd}
               (take-event events :change-plan)]
      (let [sub-id (org-sub-id db org-id)
            sub (and sub-id (get-in db [::subscriptions sub-id]))]
        (when (and sub-id
                   (= :active (:status sub))
                   (not= new-plan-id (:plan-id sub)))
          (-> db
              (update :billing/events disj cmd)
              (update :stripe/events conj
                      {:type :update-subscription
                       :subscription-id (:stripe-subscription-id sub)
                       :new-price-id (:stripe-price-id new-plan-id)})
              (assoc-in [::subscriptions sub-id :plan-id] new-plan-id)))))))

(r/defproc cancel-subscription
  (fn [{:keys [:billing/events] :as db}]
    (when-let [{:keys [org-id reason] :as cmd}
               (take-event events :cancel-subscription)]
      (let [sub-id (org-sub-id db org-id)
            sub (and sub-id (get-in db [::subscriptions sub-id]))]
        (when (and sub-id
                   (contains? #{:active :trialing} (:status sub)))
          (-> db
              (update :billing/events disj cmd)
              (update :stripe/events conj
                      {:type :cancel-subscription
                       :subscription-id (:stripe-subscription-id sub)
                       :at-period-end true})
              (update :audit/outbox conj
                      {:user-id (get-in db [::organisations org-id :owner-id])
                       :event :cancellation-requested
                       :timestamp (:clock/now db)
                       :metadata {:reason reason}})))))))

(r/defproc edit-document
  (fn [{:keys [:billing/events] :as db}]
    (when-let [{:keys [actor-id document-id new-content] :as cmd}
               (take-event events :edit-document)]
      (when (true? (get-in db [:rbac/permissions [actor-id document-id] :can-edit]))
        (-> db
            (update :billing/events disj cmd)
            (assoc-in [::documents document-id :content] new-content)))))))
```

### Library Spec Design Principles

For Recife library models:

1. Use immutable source coordinates for library model imports.
2. Expose configuration maps for provider-specific behavior.
3. Emit normalized events for all important domain transitions.
4. Keep coupling one-way: app models depend on library models, not vice versa.
5. Keep library boundaries narrow and domain-focused.

## Using These Patterns

### Composition

Combine pattern components in one run:

```clojure
(comment
  @(r/run-model
    global
    #{password-auth/register
      password-auth/login-success
      password-auth/login-failure
      password-auth/request-password-reset
      rbac/create-workspace
      rbac/add-member
      rbac/create-document
      resource-invitation/invite-to-resource
      soft-delete/delete-document
      soft-delete/restore-document
      notifications/create-mention-notification
      notifications/send-immediate-email
      usage-limits/create-document
      usage-limits/record-api-request
      comments/create-comment
      comments/notify-mentioned-user
      app-auth/create-user-on-first-login
      billing/activate-on-payment-success}))
```

Keep pattern state keys namespaced to avoid collisions.

### Adaptation

Adapt by changing:

1. State keys and event shapes to your domain language.
2. Config defaults (`::config` maps).
3. Guards and transitions to policy details.
4. Notification templates and external event contracts.
5. Invariants/properties to your explicit guarantees.

### Anti-Patterns

Avoid:

- copying provider wire formats into app-level state keys
- mixing command/event shape conventions for one concern
- adding unbounded collections without safety checks
- using implicit status transitions not represented in process code
- keeping pattern names when your domain uses different terms
