# Worked examples: from code to spec

These examples show real implementation-style code, then walk through extracting equivalent Recife model components.

## Example 1: Password Reset (Python/Flask)

```python
# models.py
class User(db.Model):
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    status = Column(Enum('active', 'locked', 'suspended'))

class PasswordResetToken(db.Model):
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('user.id'))
    status = Column(Enum('pending', 'used', 'expired'))
    expires_at = Column(DateTime)

# routes.py
@app.post('/password-reset')
def request_password_reset():
    email = request.json['email']
    user = User.query.filter_by(email=email).first()

    if not user or user.status not in ['active', 'locked']:
        return {'ok': True}

    PasswordResetToken.query.filter_by(user_id=user.id, status='pending')\
        .update({'status': 'expired'})

    token = PasswordResetToken(
        user_id=user.id,
        status='pending',
        expires_at=datetime.utcnow() + timedelta(hours=24)
    )
    db.session.add(token)
    db.session.commit()

    send_email(user.email, 'password_reset', {'token': token.id})
    return {'ok': True}
```

Key distillation notes:

- keep user status policy (`active`/`locked` can reset)
- abstract away ORM session operations
- model token lifecycle and notification intent

**Extracted Recife model (`password-reset.model.clj`):**

```clojure
(ns worked.password-reset.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::users {:u-1 {:email "ana@example.com" :status :active}}
   ::reset-tokens {}
   ::config {:token-expiry-ms (* 24 60 60 1000)}
   :auth/commands #{}
   :mail/outbox []
   :clock/now 0})

(r/defproc request-password-reset
  (fn [{:keys [:auth/commands ::users ::reset-tokens ::config :clock/now] :as db}]
    (when-let [{:keys [email]} (first commands)]
      (if-let [[user-id user] (first (filter (fn [[_ u]] (= email (:email u))) users))]
        (if (contains? #{:active :locked} (:status user))
          (let [token-id (keyword (str "prt-" (inc (count reset-tokens))))
                db' (-> db
                        (update :auth/commands disj {:email email})
                        (update ::reset-tokens
                                (fn [tokens]
                                  (into {}
                                        (map (fn [[k t]]
                                               [k (if (and (= user-id (:user-id t))
                                                           (= :pending (:status t)))
                                                    (assoc t :status :expired)
                                                    t)]))
                                        tokens)))
                        (assoc-in [::reset-tokens token-id]
                                  {:user-id user-id
                                   :status :pending
                                   :expires-at (+ now (:token-expiry-ms config))}))]
            (update db' :mail/outbox conj
                    {:to email :template :password-reset :token-id token-id}))
          db)
        ;; Mirror endpoint behavior: no-op for unknown/suspended users.
        (update db :auth/commands disj {:email email})))))

(r/defproc expire-reset-tokens
  (fn [{:keys [::reset-tokens :clock/now] :as db}]
    (reduce-kv (fn [acc token-id token]
                 (if (and (= :pending (:status token))
                          (<= (:expires-at token) now))
                   (assoc-in acc [::reset-tokens token-id :status] :expired)
                   acc))
               db
               reset-tokens)))

(rh/definvariant token-user-exists
  [{:keys [::reset-tokens ::users]}]
  (every? (fn [[_ token]] (contains? users (:user-id token))) reset-tokens))
```

## Example 2: Usage Limits (TypeScript/Node)

```typescript
// service.ts
export async function createProject(accountId: string) {
  const account = await accountRepo.get(accountId)

  if (!account) throw new Error('not-found')
  if (account.plan === 'free' && account.usage.projects >= 1) {
    throw new Error('quota-exceeded')
  }
  if (account.plan === 'pro' && account.usage.projects >= 10) {
    throw new Error('quota-exceeded')
  }

  const project = await projectRepo.create({ accountId })
  await accountRepo.incrementUsage(accountId, 'projects')

  return project
}
```

Key distillation notes:

- preserve quota policy per plan
- model usage increment transition
- model rejection path as no state transition

**Extracted Recife model (`usage-limits.model.clj`):**

```clojure
(ns worked.usage-limits.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::accounts
   {:a-free {:plan :free :usage {:projects 0} :quota {:projects 1}}
    :a-pro  {:plan :pro  :usage {:projects 0} :quota {:projects 10}}}
   ::projects {}
   :project/commands #{}})

(r/defproc create-project
  (fn [{:keys [:project/commands ::accounts ::projects] :as db}]
    (when-let [{:keys [account-id project-id]} (first commands)]
      (let [account (get accounts account-id)
            used (get-in account [:usage :projects])
            limit (get-in account [:quota :projects])]
        (if (and account (< used limit))
          (-> db
              (update :project/commands disj {:account-id account-id :project-id project-id})
              (assoc-in [::projects project-id] {:account-id account-id :status :active})
              (update-in [::accounts account-id :usage :projects] inc))
          db)))))

(rh/definvariant usage-within-quota
  [{:keys [::accounts]}]
  (every? (fn [[_ account]]
            (<= (get-in account [:usage :projects])
                (get-in account [:quota :projects])))
          accounts))

(rh/definvariant project-account-exists
  [{:keys [::projects ::accounts]}]
  (every? (fn [[_ project]]
            (contains? accounts (:account-id project)))
          projects))
```

## Example 3: Soft Delete (Java/Spring)

```java
@Service
public class DocumentService {
  public void delete(UUID id, UUID actorId) {
    Document doc = repo.findById(id).orElseThrow(NotFound::new);
    if (doc.getStatus() == Status.DELETED) {
      return;
    }
    doc.setStatus(Status.DELETED);
    doc.setDeletedAt(Instant.now());
    doc.setDeletedBy(actorId);
    repo.save(doc);
    audit.log("document_deleted", id, actorId);
  }

  public void restore(UUID id, UUID actorId) {
    Document doc = repo.findById(id).orElseThrow(NotFound::new);
    if (doc.getStatus() != Status.DELETED) {
      throw new IllegalStateException();
    }
    doc.setStatus(Status.ACTIVE);
    doc.setDeletedAt(null);
    doc.setDeletedBy(null);
    repo.save(doc);
    audit.log("document_restored", id, actorId);
  }
}
```

Key distillation notes:

- preserve state machine (`active` <-> `deleted`)
- preserve audit intent
- abstract away repository mechanics

**Extracted Recife model (`soft-delete.model.clj`):**

```clojure
(ns worked.soft-delete.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

(def global
  {::documents {:d-1 {:status :active
                      :deleted-at nil
                      :deleted-by nil}}
   :document/commands #{}
   :audit/log []
   :clock/now 0})

(r/defproc delete-document
  (fn [{:keys [:document/commands ::documents :audit/log :clock/now] :as db}]
    (when-let [{:keys [document-id actor-id]} (first commands)]
      (let [doc (get documents document-id)]
        (if (and doc (not= :deleted (:status doc)))
          (-> db
              (update :document/commands disj {:document-id document-id :actor-id actor-id})
              (assoc-in [::documents document-id :status] :deleted)
              (assoc-in [::documents document-id :deleted-at] now)
              (assoc-in [::documents document-id :deleted-by] actor-id)
              (update :audit/log conj {:event :document-deleted
                                       :document-id document-id
                                       :actor-id actor-id}))
          db)))))

(r/defproc restore-document
  (fn [{:keys [:document/commands ::documents] :as db}]
    (when-let [{:keys [restore-id actor-id]} (first commands)]
      (let [doc (get documents restore-id)]
        (if (and doc (= :deleted (:status doc)))
          (-> db
              (update :document/commands disj {:restore-id restore-id :actor-id actor-id})
              (assoc-in [::documents restore-id :status] :active)
              (assoc-in [::documents restore-id :deleted-at] nil)
              (assoc-in [::documents restore-id :deleted-by] nil)
              (update :audit/log conj {:event :document-restored
                                       :document-id restore-id
                                       :actor-id actor-id}))
          db)))))

(rh/definvariant deleted-fields-consistent
  [{:keys [::documents]}]
  (every? (fn [[_ doc]]
            (if (= :deleted (:status doc))
              (and (some? (:deleted-at doc))
                   (some? (:deleted-by doc)))
              true))
          documents))
```
