# Complete patterns

This library contains reusable Recife model patterns for common SaaS scenarios. Each pattern mirrors the original behavioral intent while expressing it as executable Clojure model code.

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
   :auth/register #{}, :auth/login #{}, :auth/request-reset #{}, :auth/complete-reset #{}
   :mail/outbox []
   :clock/now 0})

(defn user-by-email [db email]
  (first (filter (fn [[_ u]] (= email (:email u))) (::users db))))

(defn password-valid? [user password]
  (= (str "hash:" password) (:password-hash user)))

(defn token-valid? [db token]
  (and (= :pending (:status token))
       (> (:expires-at token) (:clock/now db))))

(r/defproc register
  (fn [{:keys [:auth/register] :as db}]
    (when-let [{:keys [email password]} (first register)]
      (let [exists? (some? (user-by-email db email))]
        (when (and (not exists?)
                   (>= (count password) (get-in db [::config :min-password-length])))
          (let [user-id (keyword (str "user-" (inc (count (::users db)))))
                now (:clock/now db)]
            (-> db
                (update :auth/register disj {:email email :password password})
                (assoc-in [::users user-id]
                          {:email email
                           :password-hash (str "hash:" password)
                           :status :active
                           :failed-login-attempts 0
                           :locked-until nil
                           :created-at now})
                (update :mail/outbox conj {:to email :template :welcome}))))))))

(r/defproc login-success
  (fn [{:keys [:auth/login] :as db}]
    (when-let [{:keys [email password]} (first login)]
      (when-let [[user-id user] (user-by-email db email)]
        (let [now (:clock/now db)
              locked? (and (= :locked (:status user))
                           (some? (:locked-until user))
                           (> (:locked-until user) now))]
          (when (and (not locked?)
                     (password-valid? user password)
                     (not= :deactivated (:status user)))
            (let [session-id (keyword (str "session-" (inc (count (::sessions db)))))
                  db' (-> db
                          (update :auth/login disj {:email email :password password})
                          (assoc-in [::users user-id :status] :active)
                          (assoc-in [::users user-id :failed-login-attempts] 0)
                          (assoc-in [::users user-id :locked-until] nil)
                          (assoc-in [::sessions session-id]
                                    {:user-id user-id
                                     :created-at now
                                     :expires-at (+ now (get-in db [::config :session-duration-ms]))
                                     :status :active}))]
              db')))))))

(r/defproc login-failure
  (fn [{:keys [:auth/login] :as db}]
    (when-let [{:keys [email password] :as cmd} (first login)]
      (when-let [[user-id user] (user-by-email db email)]
        (when (not (password-valid? user password))
          (let [attempts (inc (:failed-login-attempts user))
                max-attempts (get-in db [::config :max-login-attempts])]
            (-> db
                (update :auth/login disj cmd)
                (assoc-in [::users user-id :failed-login-attempts] attempts)
                (cond-> (>= attempts max-attempts)
                  (assoc-in [::users user-id :status] :locked)
                  (assoc-in [::users user-id :locked-until]
                            (+ (:clock/now db)
                               (get-in db [::config :lockout-duration-ms])))))))))))

(r/defproc request-password-reset
  (fn [{:keys [:auth/request-reset] :as db}]
    (when-let [{:keys [email]} (first request-reset)]
      (when-let [[user-id user] (user-by-email db email)]
        (when (contains? #{:active :locked} (:status user))
          (let [token-id (keyword (str "token-" (inc (count (::reset-tokens db)))))
                now (:clock/now db)
                db' (-> db
                        (update :auth/request-reset disj {:email email})
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
                    {:to email
                     :template :password-reset
                     :data {:token-id token-id}})))))))

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
    (reduce-kv (fn [acc token-id token]
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
  (:require [recife.core :as r]
            [clojure.set :as set]))

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
   ::users {}
   ::workspaces {}
   ::workspace-memberships {}
   ::documents {}
   ::document-views []
   :rbac/events #{}})

(defn effective-permissions [db role-id]
  (let [role (get-in db [::roles role-id])]
    (if-let [parent (:inherits-from role)]
      (set/union (:permissions role)
                 (effective-permissions db parent))
      (:permissions role))))

(defn can-read? [db workspace-id user-id]
  (contains? (effective-permissions db (get-in db [::workspace-memberships [workspace-id user-id] :role]))
             "documents.read"))

(defn can-write? [db workspace-id user-id]
  (contains? (effective-permissions db (get-in db [::workspace-memberships [workspace-id user-id] :role]))
             "documents.write"))

(r/defproc view-document
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id document-id] :as cmd} (first events)]
      (let [workspace-id (get-in db [::documents document-id :workspace-id])]
        (when (can-read? db workspace-id user-id)
          (-> db
              (update :rbac/events disj cmd)
              (update ::document-views conj {:user-id user-id :document-id document-id :at (:clock/now db)})))))))

(r/defproc edit-document
  (fn [{:keys [:rbac/events] :as db}]
    (when-let [{:keys [user-id document-id new-content] :as cmd} (first events)]
      (let [workspace-id (get-in db [::documents document-id :workspace-id])]
        (when (can-write? db workspace-id user-id)
          (-> db
              (update :rbac/events disj cmd)
              (assoc-in [::documents document-id :content] new-content)))))))
```

## Pattern 3: Invitation to Resource

File: `resource-invitation.model.clj`

```clojure
(ns patterns.resource-invitation.model
  (:require [recife.core :as r]))

(def global
  {::config {:invitation-expiry-ms (* 7 24 60 60 1000)}
   ::resources {}
   ::resource-shares {}
   ::resource-invitations {}
   :resource/events #{}
   :mail/outbox []
   :clock/now 0})

(defn can-invite? [db resource-id user-id]
  (let [resource (get-in db [::resources resource-id])
        share (get-in db [::resource-shares [resource-id user-id]])]
    (or (= user-id (:owner-id resource))
        (contains? #{:edit :admin} (:permission share)))))

(defn invitation-valid? [db invitation]
  (and (= :pending (:status invitation))
       (> (:expires-at invitation) (:clock/now db))))

(r/defproc invite-to-resource
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [inviter-id resource-id email permission] :as cmd} (first events)]
      (let [resource (get-in db [::resources resource-id])
            owner? (= inviter-id (:owner-id resource))
            existing-share? (some (fn [[[_ uid] share]]
                                    (and (= resource-id (:resource-id share))
                                         (= email (:email (get-in db [::users uid])))))
                                  (::resource-shares db))
            existing-invitation (first (filter (fn [[_ inv]]
                                                 (and (= resource-id (:resource-id inv))
                                                      (= email (:email inv))))
                                               (::resource-invitations db)))]
        (when (and (can-invite? db resource-id inviter-id)
                   (or (contains? #{:view :edit} permission)
                       (and (= :admin permission) owner?))
                   (not existing-share?)
                   (or (nil? existing-invitation)
                       (not (invitation-valid? db (val existing-invitation)))))
          (let [inv-id (keyword (str "inv-" (inc (count (::resource-invitations db)))))
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
                (update :mail/outbox conj {:to email :template :resource-invitation}))))))))

(r/defproc accept-invitation
  (fn [{:keys [:resource/events] :as db}]
    (when-let [{:keys [user-id invitation-id] :as cmd} (first events)]
      (when-let [inv (get-in db [::resource-invitations invitation-id])]
        (when (and (invitation-valid? db inv)
                   (= (:email inv) (get-in db [::users user-id :email])))
          (-> db
              (update :resource/events disj cmd)
              (assoc-in [::resource-invitations invitation-id :status] :accepted)
              (assoc-in [::resource-shares [(:resource-id inv) user-id]]
                        {:resource-id (:resource-id inv)
                         :user-id user-id
                         :permission (:permission inv)
                         :status :active
                         :created-at (:clock/now db)})))))))

(r/defproc invitation-expires
  (fn [{:keys [::resource-invitations :clock/now] :as db}]
    (reduce-kv (fn [acc invitation-id inv]
                 (if (and (= :pending (:status inv))
                          (<= (:expires-at inv) now))
                   (assoc-in acc [::resource-invitations invitation-id :status] :expired)
                   acc))
               db
               resource-invitations)))
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

(defn can-admin? [db workspace-id user-id]
  (true? (get-in db [::workspace-memberships [workspace-id user-id] :can-admin])))

(defn can-restore? [db document]
  (and (= :deleted (:status document))
       (> (+ (:deleted-at document) (get-in db [::config :retention-period-ms]))
          (:clock/now db))))

(r/defproc delete-document
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first events)]
      (when-let [doc (get-in db [::documents document-id])]
        (when (and (= :active (:status doc))
                   (or (= actor-id (:created-by doc))
                       (can-admin? db (:workspace-id doc) actor-id)))
          (-> db
              (update :document/events disj cmd)
              (assoc-in [::documents document-id :status] :deleted)
              (assoc-in [::documents document-id :deleted-at] (:clock/now db))
              (assoc-in [::documents document-id :deleted-by] actor-id)))))))

(r/defproc restore-document
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first events)]
      (when-let [doc (get-in db [::documents document-id])]
        (when (and (can-restore? db doc)
                   (or (= actor-id (:deleted-by doc))
                       (can-admin? db (:workspace-id doc) actor-id)))
          (-> db
              (update :document/events disj cmd)
              (assoc-in [::documents document-id :status] :active)
              (assoc-in [::documents document-id :deleted-at] nil)
              (assoc-in [::documents document-id :deleted-by] nil)))))))

(r/defproc permanently-delete
  (fn [{:keys [:document/events] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first events)]
      (when-let [doc (get-in db [::documents document-id])]
        (when (and (= :deleted (:status doc))
                   (can-admin? db (:workspace-id doc) actor-id))
          (-> db
              (update :document/events disj cmd)
              (update ::documents dissoc document-id)))))))

(r/defproc retention-expires
  (fn [{:keys [::documents :clock/now] :as db}]
    (reduce-kv (fn [acc document-id doc]
                 (let [deadline (+ (:deleted-at doc) (get-in db [::config :retention-period-ms]))]
                   (if (and (= :deleted (:status doc))
                            (<= deadline now))
                     (update acc ::documents dissoc document-id)
                     acc)))
               db
               documents)))
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

(defn immediate-email? [settings kind]
  (= :immediately
     (case kind
       :MentionNotification (:email-on-mention settings)
       :ReplyNotification (:email-on-comment settings)
       :ShareNotification (:email-on-share settings)
       :AssignmentNotification (:email-on-assignment settings)
       :never)))

(r/defproc create-mention-notification
  (fn [{:keys [:notification/events] :as db}]
    (when-let [{:keys [user-id comment-id mentioned-by] :as cmd} (first events)]
      (let [nid (keyword (str "notif-" (inc (count (::notifications db)))))
            settings (get-in db [::notification-settings user-id])
            db' (-> db
                    (update :notification/events disj cmd)
                    (assoc-in [::notifications nid]
                              {:user-id user-id
                               :kind :MentionNotification
                               :comment-id comment-id
                               :mentioned-by mentioned-by
                               :created-at (:clock/now db)
                               :status :unread
                               :email-status :pending}) )]
        (if (immediate-email? settings :MentionNotification)
          (-> db'
              (assoc-in [::notifications nid :email-status] :sent)
              (update :mail/outbox conj {:to (get-in db [::users user-id :email])
                                         :template :mention-notification
                                         :data {:comment-id comment-id}}))
          db')))))

(r/defproc process-digests
  (fn [{:keys [::users ::notifications :clock/now] :as db}]
    (reduce-kv (fn [acc user-id user]
                 (let [settings (get-in acc [::notification-settings user-id])
                       due? (and (:digest-enabled settings)
                                 (<= (get-in acc [::users user-id :next-digest-at] Long/MAX_VALUE)
                                     now))
                       pending (filter (fn [[_ n]]
                                         (and (= user-id (:user-id n))
                                              (= :pending (:email-status n))
                                              (>= (:created-at n) (- now (* 24 60 60 1000)))))
                                       (::notifications acc))]
                   (if (and due? (seq pending))
                     (let [batch-id (keyword (str "digest-" (inc (count (::digest-batches acc)))))]
                       (-> acc
                           (assoc-in [::digest-batches batch-id]
                                     {:user-id user-id
                                      :notification-ids (mapv first pending)
                                      :created-at now})
                           (reduce (fn [x [nid _]]
                                     (assoc-in x [::notifications nid :email-status] :digested))
                                   pending)
                           (update :mail/outbox conj {:to (:email user)
                                                      :template :daily-digest
                                                      :data {:batch-id batch-id}})))
                     acc)))
               db
               users)))
```

## Pattern 6: Usage Limits & Quotas

File: `usage-limits.model.clj`

```clojure
(ns patterns.usage-limits.model
  (:require [recife.core :as r]))

(def global
  {::plans {}
   ::workspaces {}
   ::documents {}
   ::workspace-memberships {}
   ::workspace-usage {}
   ::usage-events []
   :usage/events #{}, :mail/outbox []
   :clock/now 0})

(defn unlimited? [v] (nil? v))

(defn can-add-document? [db workspace-id]
  (let [plan (get-in db [::plans (get-in db [::workspaces workspace-id :plan-id])])
        max-docs (:max-documents plan)
        count-docs (->> (::documents db) vals (filter #(= workspace-id (:workspace-id %))) count)]
    (or (unlimited? max-docs) (< count-docs max-docs))))

(r/defproc create-document
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id title content] :as cmd} (first events)]
      (when (can-add-document? db workspace-id)
        (let [doc-id (keyword (str "doc-" (inc (count (::documents db)))))
              usage-id (keyword (str workspace-id "-usage"))]
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::documents doc-id]
                        {:workspace-id workspace-id :created-by user-id :title title :content content})
              (update ::usage-events conj {:workspace-id workspace-id :type :document-created :amount 1 :recorded-at (:clock/now db)})
              (update-in [::workspace-usage usage-id :storage-bytes-used] + (count content))))))))

(r/defproc add-member
  (fn [{:keys [:usage/events] :as db}]
    (when-let [{:keys [workspace-id user-id role] :as cmd} (first events)]
      (let [plan (get-in db [::plans (get-in db [::workspaces workspace-id :plan-id])])
            max-members (:max-team-members plan)
            member-count (->> (::workspace-memberships db) keys (filter #(= workspace-id (first %))) count)]
        (when (or (unlimited? max-members) (< member-count max-members))
          (-> db
              (update :usage/events disj cmd)
              (assoc-in [::workspace-memberships [workspace-id user-id]] {:role role})
              (update ::usage-events conj {:workspace-id workspace-id :type :member-added :amount 1 :recorded-at (:clock/now db)})))))))

(r/defproc reset-daily-api-usage
  (fn [{:keys [::workspace-usage :clock/now] :as db}]
    (reduce-kv (fn [acc usage-id usage]
                 (if (<= (:next-reset-at usage) now)
                   (-> acc
                       (assoc-in [::workspace-usage usage-id :api-requests-today] 0)
                       (assoc-in [::workspace-usage usage-id :next-reset-at] (+ now (* 24 60 60 1000))))
                   acc))
               db
               workspace-usage)))
```

## Pattern 7: Comments with Mentions

File: `comments.model.clj`

```clojure
(ns patterns.comments.model
  (:require [clojure.string :as str]
            [recife.core :as r]))

(def global
  {::comments {}
   ::comment-mentions {}
   ::comment-reactions {}
   :comments/events #{}
   :notifications/outbox []
   :clock/now 0})

(defn parse-mentions [body]
  (->> (str/split body #"\\s+")
       (filter #(str/starts-with? % "@"))
       (map #(keyword (subs % 1)))
       set))

(r/defproc create-comment
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [author-id parent-id body] :as cmd} (first events)]
      (let [comment-id (keyword (str "comment-" (inc (count (::comments db)))))
            mentions (parse-mentions body)
            now (:clock/now db)
            db' (-> db
                    (update :comments/events disj cmd)
                    (assoc-in [::comments comment-id]
                              {:parent-id parent-id
                               :reply-to nil
                               :author-id author-id
                               :body body
                               :created-at now
                               :edited-at nil
                               :status :active})
                    (reduce (fn [acc mentioned-user]
                              (let [mid [comment-id mentioned-user]]
                                (assoc-in acc [::comment-mentions mid]
                                          {:comment-id comment-id
                                           :user-id mentioned-user
                                           :notified false})))
                            mentions))]
        (reduce (fn [acc mentioned-user]
                  (update acc :notifications/outbox conj
                          {:to mentioned-user :type :mention :comment-id comment-id}))
                db'
                mentions)))))

(r/defproc reply-to-comment
  (fn [{:keys [:comments/events] :as db}]
    (when-let [{:keys [author-id parent-id reply-to body] :as cmd} (first events)]
      (when-let [parent-comment (get-in db [::comments reply-to])]
        (when (= :active (:status parent-comment))
          (let [comment-id (keyword (str "comment-" (inc (count (::comments db)))))
                now (:clock/now db)]
            (-> db
                (update :comments/events disj cmd)
                (assoc-in [::comments comment-id]
                          {:parent-id parent-id
                           :reply-to reply-to
                           :author-id author-id
                           :body body
                           :created-at now
                           :edited-at nil
                           :status :active}))))))))
```

## Pattern 8: Integrating Library Specs

In Recife, this means composing reusable library model namespaces with application model namespaces.

### Example: OAuth Authentication

Files:

- `oauth2/model.clj` (library model)
- `app-auth/model.clj` (application model)

```clojure
(ns app-auth.model
  (:require [recife.core :as r]
            [oauth2.model :as oauth]))

(def global
  (merge oauth/global
         {::users {}
          ::user-preferences {}
          :app/events #{}}))

(r/defproc create-user-on-first-login
  (fn [{:keys [:oauth/events] :as db}]
    (when-let [event (first (filter #(= :authentication-succeeded (:type %)) events))]
      (when-not (some (fn [[_ user]] (= (:email event) (:email user))) (::users db))
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
              (assoc-in [:oauth/sessions (:session-id event) :user-id] user-id)))))))
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
          :mail/outbox []
          :billing/events #{}}))

(r/defproc activate-on-payment-success
  (fn [{:keys [:stripe/events] :as db}]
    (when-let [event (first (filter #(= :payment-succeeded (:type %)) events))]
      (let [org-id (first (keep (fn [[oid org]] (when (= (:customer-id event) (:stripe-customer-id org)) oid))
                               (::organisations db)))
            sub-id (some->> (::subscriptions db)
                            (keep (fn [[sid sub]] (when (= org-id (:organisation-id sub)) sid)))
                            first)]
        (when (and org-id sub-id
                   (contains? #{:trialing :past-due} (get-in db [::subscriptions sub-id :status])))
          (-> db
              (update :stripe/events disj event)
              (assoc-in [::subscriptions sub-id :status] :active)
              (assoc-in [::subscriptions sub-id :current-period-ends-at] (:period-end event))
              (update :mail/outbox conj
                      {:to (get-in db [::organisations org-id :owner-email])
                       :template :payment-confirmed
                       :data {:amount (:amount event)
                              :next-billing (:period-end event)}})))))))
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
                #{password-auth/register
                  password-auth/login-success
                  rbac/view-document
                  resource-invitation/invite-to-resource
                  soft-delete/delete-document
                  notifications/process-digests
                  usage-limits/create-document
                  comments/create-comment}))
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
