---
name: distill
description: This skill should be used when the user has "existing code" and wants to "extract a model", "distill behaviour from code", "reverse engineer a specification", or wants to produce a Recife model from an existing codebase.
---

# Distillation guide

This guide covers extracting Recife specifications from existing codebases. The core challenge is the same as forward elicitation: finding the right level of abstraction. In elicitation you filter out implementation ideas as they arise. In distillation you filter out implementation details that already exist. Both require the same judgement about what matters at the domain level.

Code tells you *how* something works. A specification captures *what* it does and *why* it matters. The skill is asking "why does the stakeholder care about this?" and "could this be different while still being the same system?"

## Scoping the distillation effort

Before diving into code, establish what you are trying to specify. Not every line of code deserves a place in the spec.

### Questions to ask first

1. **"What subset of this codebase are we specifying?"**
   Mono repos often contain multiple distinct systems. You may only need a spec for one service or domain. Clarify boundaries explicitly before starting.

2. **"Is there code we should deliberately exclude?"**
   - **Legacy code**: features kept for backwards compatibility but not part of the core system
   - **Incidental code**: supporting infrastructure that is not domain-level (logging, metrics, deployment)
   - **Deprecated paths**: code scheduled for removal
   - **Experimental features**: behind feature flags, not yet design decisions

3. **"Who owns this spec?"**
   Different teams may own different parts of a mono repo. Each team's spec should focus on their domain.

### The "Would we rebuild this?" test

For any code path you encounter, ask: "If we rebuilt this system from scratch, would this be in the requirements?"

- Yes: include in spec
- No, it is legacy: exclude
- No, it is infrastructure: exclude
- No, it is a workaround: exclude (but note the underlying need it addresses)

### Documenting scope decisions

At the top of a distilled model file, document what is included and excluded:

```clojure
(ns interview-scheduling.model
  (:require [recife.core :as r]
            [recife.helpers :as rh]))

;; Scope: Interview scheduling flow only
;; Includes: candidacy, interview, interview-slot, invitation, feedback
;; Excludes:
;;   - user authentication (use auth model)
;;   - analytics/reporting (separate model)
;;   - legacy v1 api (deprecated, not specified)
;;   - greenhouse sync (use greenhouse model)
```

Use a namespace declaration as the first form in every Recife `.clj` model file, then capture scope notes as comments near the top.

## Finding the right level of abstraction

Distillation and elicitation share the same fundamental challenge: choosing what to include. The tests below work in both directions, whether you are hearing a stakeholder describe a feature or reading code that implements it.

### The "Why" test

For every detail in the code, ask: "Why does the stakeholder care about this?"

| Code detail | Why? | Include? |
|-------------|------|----------|
| Invitation expires in 7 days | Affects candidate experience | Yes |
| Token is 32 bytes URL-safe | Security implementation | No |
| Sessions stored in Redis | Performance choice | No |
| Uses PostgreSQL JSONB | Database implementation | No |
| Slot status changes to 'proposed' | Affects what candidate sees | Yes |
| Email sent when invitation accepted | Communication requirement | Yes |

If you cannot articulate why a stakeholder would care, it is probably implementation.

### The "Could it be different?" test

Ask: "Could this be implemented differently while still being the same system?"

- If yes: probably implementation detail, abstract it away
- If no: probably domain-level, include it

| Detail | Could be different? | Include? |
|--------|---------------------|----------|
| `secrets.token_urlsafe(32)` | Yes, any secure token generation | No |
| 7-day invitation expiry | No, this is the design decision | Yes |
| PostgreSQL database | Yes, any database | No |
| "Pending, Confirmed, Completed" states | No, this is the workflow | Yes |

### The "Template vs Instance" test

Is this a **category** of thing, or a **specific instance**?

| Instance (often implementation) | Template (often domain-level) |
|--------------------------------|-------------------------------|
| Google OAuth | Authentication provider |
| Slack webhook | Notification channel |
| SendGrid API | Email delivery |
| `timedelta(hours=3)` | Confirmation deadline |

Sometimes the instance IS the domain concern. See "The concrete detail problem" below.

## The distillation mindset

### Code is over-specified

Every line of code makes decisions that might not matter at the domain level:

```python
# Code tells you:
def send_invitation(candidate_id: int, slot_ids: List[int]) -> Invitation:
    candidate = db.session.query(Candidate).get(candidate_id)
    slots = db.session.query(InterviewSlot).filter(
        InterviewSlot.id.in_(slot_ids),
        InterviewSlot.status == 'confirmed'
    ).all()

    invitation = Invitation(
        candidate_id=candidate_id,
        token=secrets.token_urlsafe(32),
        expires_at=datetime.utcnow() + timedelta(days=7),
        status='pending'
    )
    db.session.add(invitation)

    for slot in slots:
        slot.status = 'proposed'
        invitation.slots.append(slot)

    db.session.commit()

    send_email(
        to=candidate.email,
        template='interview_invitation',
        context={'invitation': invitation, 'slots': slots}
    )

    return invitation
```

```clojure
;; Model should say:
(r/defproc send-invitation
  (fn [{:keys [::candidacies ::slots ::config :mail/outbox :commands/send-invitation] :as db}]
    (when-let [{:keys [candidacy-id slot-ids]} (first send-invitation)]
      (let [candidacy (get candidacies candidacy-id)
            selected-slots (select-keys slots slot-ids)]
        (when (every? (fn [[_ slot]] (= :confirmed (:status slot))) selected-slots)
          (let [invitation-id (keyword (str "inv-" (inc (count (::invitations db)))))
                expires-at (+ (:clock/now db) (:invitation-expiry-ms config))
                db' (-> db
                        (update :commands/send-invitation disj {:candidacy-id candidacy-id :slot-ids slot-ids})
                        (assoc-in [::invitations invitation-id]
                                  {:candidacy-id candidacy-id
                                   :slot-ids slot-ids
                                   :expires-at expires-at
                                   :status :pending}))
                db'' (reduce (fn [acc [slot-id _]]
                               (assoc-in acc [::slots slot-id :status] :proposed))
                             db'
                             selected-slots)]
            (update db'' :mail/outbox conj
                    {:to (get-in candidacy [:candidate :email])
                     :template :interview-invitation
                     :invitation-id invitation-id})))))))
```

What we dropped:
- `candidate_id: int` became a model-level `candidacy-id`
- `db.session.query(...)` became map/set lookups in model state
- `secrets.token_urlsafe(32)` removed entirely (token generation is implementation)
- `datetime.utcnow() + timedelta(...)` became model time arithmetic (`:clock/now` + config)
- `db.session.add/commit` became explicit state transition in one process step
- `invitation.slots.append(slot)` became slot-id membership/state updates

### Ask "Would a product owner care?"

For every detail in the code, ask:

| Code detail | Product owner cares? | Include? |
|-------------|---------------------|----------|
| Invitation expires in 7 days | Yes, affects candidate experience | Yes |
| Token is 32 bytes URL-safe | No, security implementation | No |
| Uses SQLAlchemy ORM | No, persistence mechanism | No |
| Email template name | Maybe, if templates are design decisions | Maybe |
| Slot status changes to 'proposed' | Yes, affects what candidate sees | Yes |
| Database transaction commits | No, implementation detail | No |

### Distinguish means from ends

**Means:** how the code achieves something.
**Ends:** what outcome the system needs.

| Means (code) | Ends (model) |
|--------------|-------------|
| `requests.post('https://slack.com/api/...')` | `(update db :notifications/outbox conj {:channel :slack ...})` |
| `candidate.oauth_token = google.exchange(code)` | `candidate state transitions to :authenticated` |
| `redis.setex(f'session:{id}', 86400, data)` | `(assoc-in db [::sessions sid :expires-at] (+ now 86400000))` |
| `for slot in slots: slot.status = 'cancelled'` | `(reduce (fn [acc sid] (assoc-in acc [::slots sid :status] :cancelled)) db slot-ids)` |

## The concrete detail problem

The hardest judgement call: when is a concrete detail part of the domain vs just implementation?

### Google OAuth example

You find this code:
```python
OAUTH_PROVIDERS = {
    'google': GoogleOAuthProvider(client_id=..., client_secret=...),
}

def authenticate(provider: str, code: str) -> User:
    return OAUTH_PROVIDERS[provider].authenticate(code)
```

**Question:** Is "Google OAuth" domain-level or implementation?

**It is implementation if:**
- Google is just the auth mechanism chosen
- It could be replaced with any OAuth provider
- Users do not see or care which provider
- The code is written generically (provider is a parameter)

**It is domain-level if:**
- Users explicitly choose Google (vs Microsoft, etc.)
- "Sign in with Google" is a feature
- Google-specific scopes or permissions are used
- Multiple providers are supported as a feature

**How to tell:** Look at the UI and user flows. If users see "Sign in with Google" as a choice, it is domain-level. If they just see "Sign in" and Google happens to be behind it, it is implementation.

### Database choice example

You find PostgreSQL-specific code:
```python
from sqlalchemy.dialects.postgresql import JSONB, ARRAY

class Candidate(Base):
    skills = Column(ARRAY(String))
    metadata = Column(JSONB)
```

**Almost always implementation.** The model should say:
```clojure
(def global
  {::candidates
   {:candidate-1 {:skills #{\"clojure\" \"tla+\"}
                  :metadata nil}}})
```

The specific database is rarely domain-level. Exception: if the system explicitly promises PostgreSQL compatibility or specific PostgreSQL features to users.

### Third-party integration example

You find Greenhouse ATS integration:
```python
class GreenhouseSync:
    def import_candidate(self, greenhouse_id: str) -> Candidate:
        data = self.client.get_candidate(greenhouse_id)
        return Candidate(
            name=data['name'],
            email=data['email'],
            greenhouse_id=greenhouse_id,
            source='greenhouse'
        )
```

**Could be either:**

**Implementation if:**
- Greenhouse is just where candidates happen to come from
- Could be swapped for Lever, Workable, etc.
- The integration is an implementation detail of "candidates are imported"

Model:
```clojure
(def global
  {::candidates {}
   :greenhouse/inbound #{}})

;; Candidate source is tracked as a plain field in model state.
;; e.g. {:name \"Ana\" :email \"ana@example.com\" :source :greenhouse}
```

**Product-level if:**
- "Greenhouse integration" is a selling point
- Users configure their Greenhouse connection
- Greenhouse-specific features are exposed (like syncing feedback back)

Model:
```clojure
(r/defproc sync-from-greenhouse
  (fn [{:keys [:greenhouse/inbound] :as db}]
    (when-let [candidate-data (first inbound)]
      (-> db
          (update :greenhouse/inbound disj candidate-data)
          (assoc-in [::candidates (keyword (:id candidate-data))]
                    {:name (:name candidate-data)
                     :email (:email candidate-data)
                     :greenhouse-id (:id candidate-data)})))))
```

### The "Multiple implementations" heuristic

Look for variation in the codebase:

- If there is only one OAuth provider, probably implementation
- If there are multiple OAuth providers, probably domain-level
- If there is only one notification channel, probably implementation
- If there are Slack AND email AND SMS, probably domain-level

The presence of multiple implementations suggests the variation itself is a domain concern.

## Distillation process

### Step 1: Map the territory

Before extracting any specification, understand the codebase structure:

1. **Identify entry points.** API routes, CLI commands, message handlers, scheduled jobs.
2. **Find the domain models.** Usually in `models/`, `entities/`, `domain/`.
3. **Locate business logic.** Services, use cases, handlers.
4. **Note external integrations.** What third parties does it talk to?

Create a rough map:
```
Entry points:
  - API: /api/candidates/*, /api/interviews/*, /api/invitations/*
  - Webhooks: /webhooks/greenhouse, /webhooks/calendar
  - Jobs: send_reminders, expire_invitations, sync_calendars

Models:
  - Candidate, Interview, InterviewSlot, Invitation, Feedback

Services:
  - SchedulingService, NotificationService, CalendarService

Integrations:
  - Google Calendar, Slack, Greenhouse, SendGrid
```

### Step 2: Extract entity states

Look at enum fields and status columns:

```python
class Invitation(Base):
    status = Column(Enum('pending', 'accepted', 'declined', 'expired'))
```

Becomes:
```clojure
(def global
  {::invitations
   {:inv-1 {:status :pending}
    :inv-2 {:status :accepted}
    :inv-3 {:status :declined}
    :inv-4 {:status :expired}}})
```

Look for enum definitions, status or state columns, constants like `STATUS_PENDING = 'pending'`, and state machine libraries (e.g. `transitions`, `django-fsm`).

### Step 3: Extract transitions

Find where status changes happen:

```python
def accept_invitation(invitation_id: int, slot_id: int):
    invitation = get_invitation(invitation_id)

    if invitation.status != 'pending':
        raise InvalidStateError()
    if invitation.expires_at < datetime.utcnow():
        raise ExpiredError()

    slot = get_slot(slot_id)
    if slot not in invitation.slots:
        raise InvalidSlotError()

    invitation.status = 'accepted'
    slot.status = 'booked'

    # Release other slots
    for other_slot in invitation.slots:
        if other_slot.id != slot_id:
            other_slot.status = 'available'

    # Create the interview
    interview = Interview(
        candidate_id=invitation.candidate_id,
        slot_id=slot_id,
        status='scheduled'
    )

    notify_interviewers(interview)
    send_confirmation_email(invitation.candidate, interview)
```

Extract:
```clojure
(r/defproc candidate-accepts-invitation
  (fn [{:keys [::invitations ::slots ::interviews :notifications/outbox :mail/outbox
               :commands/candidate-accepts :clock/now] :as db}]
    (when-let [{:keys [invitation-id slot-id]} (first candidate-accepts)]
      (let [invitation (get invitations invitation-id)
            slot (get slots slot-id)]
        (when (and (= :pending (:status invitation))
                   (> (:expires-at invitation) now)
                   (contains? (set (:slot-ids invitation)) slot-id))
          (let [other-slot-ids (disj (set (:slot-ids invitation)) slot-id)
                db' (-> db
                        (update :commands/candidate-accepts disj {:invitation-id invitation-id :slot-id slot-id})
                        (assoc-in [::invitations invitation-id :status] :accepted)
                        (assoc-in [::slots slot-id :status] :booked))
                db'' (reduce (fn [acc sid]
                               (assoc-in acc [::slots sid :status] :available))
                             db'
                             other-slot-ids)
                interview-id (keyword (str "int-" (inc (count interviews))))]
            (-> db''
                (assoc-in [::interviews interview-id]
                          {:candidacy-id (:candidacy-id invitation)
                           :slot-id slot-id
                           :status :scheduled})
                (update :notifications/outbox conj {:to (:interviewer-ids slot)
                                                    :kind :interview-scheduled})
                (update :mail/outbox conj {:to (:candidate-email invitation)
                                           :template :invitation-accepted}))))))))
```

**Key extraction patterns:**

| Code pattern | Recife model pattern |
|--------------|----------------------|
| `if x.status != 'pending': raise` | guard `when (= :pending (:status x))` |
| `if x.expires_at < now: raise` | guard `when (> (:expires-at x) now)` |
| `if item not in collection: raise` | guard `when (contains? (set coll) item)` |
| `x.status = 'accepted'` | `(assoc-in db [... :status] :accepted)` |
| `Model.create(...)` | `(assoc-in db [::model id] {...})` |
| `send_email(...)` | `(update db :mail/outbox conj {...})` |
| `notify(...)` | `(update db :notifications/outbox conj {...})` |

### Step 4: Find temporal triggers

Look for scheduled jobs and time-based logic:

```python
# In celery tasks or cron jobs
@app.task
def expire_invitations():
    expired = Invitation.query.filter(
        Invitation.status == 'pending',
        Invitation.expires_at < datetime.utcnow()
    ).all()

    for invitation in expired:
        invitation.status = 'expired'
        for slot in invitation.slots:
            slot.status = 'available'
        notify_candidate_expired(invitation)

@app.task
def send_reminders():
    upcoming = Interview.query.filter(
        Interview.status == 'scheduled',
        Interview.slot.time.between(
            datetime.utcnow() + timedelta(hours=1),
            datetime.utcnow() + timedelta(hours=2)
        )
    ).all()

    for interview in upcoming:
        send_reminder_notification(interview)
```

Extract:
```clojure
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

(r/defproc interview-reminder
  (fn [{:keys [::interviews :clock/now] :as db}]
    (reduce-kv (fn [acc _ interview]
                 (if (and (= :scheduled (:status interview))
                          (<= (- (:slot-time interview) (* 60 60 1000)) now))
                   (update acc :notifications/outbox conj
                           {:to (:interviewer-ids interview)
                            :template :reminder})
                   acc))
               db
               interviews)))
```

### Step 5: Identify external boundaries

Look for third-party API calls, webhook handlers, import/export functions, and data that is read but never written (or vice versa).

These often indicate external entities:

```python
# Candidate data comes from Greenhouse, we don't create it
def import_from_greenhouse(webhook_data):
    candidate = Candidate.query.filter_by(
        greenhouse_id=webhook_data['id']
    ).first()

    if not candidate:
        candidate = Candidate(greenhouse_id=webhook_data['id'])

    candidate.name = webhook_data['name']
    candidate.email = webhook_data['email']
```

Suggests:
```clojure
;; External boundary represented as inbound payloads.
(def global
  {:greenhouse/inbound #{}           ;; raw webhook events
   ::candidates {}})                 ;; normalized domain projection
```

### Step 6: Abstract away implementation

Now make a pass through your extracted spec and remove implementation details.

**Before (too concrete):**
```clojure
;; Too concrete: persistence/storage details leaking into the model.
{:candidate-id 123
 :token "K2m2zN6u..."
 :created-at #inst "2026-02-10T10:00:00.000-00:00"
 :expires-at #inst "2026-02-17T10:00:00.000-00:00"
 :status :pending}
```

**After (domain-level):**
```clojure
{:candidacy-id :cand-42
 :created-at 1739181600000
 :expires-at 1739786400000
 :status :pending}

;; Derived check used by processes/properties:
(defn expired? [invitation now]
  (<= (:expires-at invitation) now))
```

Changes:
- `candidate-id` remains an identifier in model state, while relationship semantics are carried by process usage
- `token` removed (implementation concern)
- wall-clock types became model-time numbers/timestamps
- added a derived helper (`expired?`) for clarity

### Step 7: Validate with stakeholders

The extracted spec is a hypothesis. Validate it:

1. **Show the spec to the original developers.** "Is this what the system does?"
2. **Show to stakeholders.** "Is this what the system should do?"
3. **Look for gaps.** Code often has bugs or missing features; the spec might reveal them.

Common findings:
- "Oh, that retry logic was a hack, we should remove it"
- "Actually we wanted X but never built it"
- "These two code paths should be the same but aren't"

## Recognising library model candidates

During distillation, stay alert for code that implements **generic integration patterns** rather than application-specific logic. These belong in library models, not your main specification.

The same principle applies in elicitation. When a stakeholder describes "we use Google for login" or "payments go through Stripe", pause and consider whether this is a library model.

### Signals in the code

**Third-party integration modules:**
```python
# Finding code like this suggests a library model
class StripeWebhookHandler:
    def handle_invoice_paid(self, event):
        ...
    def handle_subscription_cancelled(self, event):
        ...

class GoogleOAuthProvider:
    def exchange_code(self, code):
        ...
    def refresh_token(self, refresh_token):
        ...
```

**Generic patterns with specific providers:**
- OAuth flows (Google, Microsoft, GitHub)
- Payment processing (Stripe, PayPal)
- Email delivery (SendGrid, Postmark, SES)
- Calendar sync (Google Calendar, Outlook)
- ATS integrations (Greenhouse, Lever)
- File storage (S3, GCS)

**Configuration-driven integrations:**
```python
# Heavy configuration suggests the integration itself is separable
OAUTH_CONFIG = {
    'google': {'client_id': ..., 'scopes': ...},
    'microsoft': {'client_id': ..., 'scopes': ...},
}
```

### Questions to ask

1. **"Is this integration logic, or application logic?"**
   Integration: how to talk to Stripe.
   Application: what to do when payment succeeds.

2. **"Would another application integrate the same way?"**
   If yes, library model candidate. If no, probably application-specific.

3. **"Does the code separate integration from application concerns?"**
   If cleanly separated, easy to extract to library model. If tangled, might need refactoring first (but the spec should still separate them).

### How to handle

**Option 1: Reference an existing library model**

If a standard library model exists for this integration:
```clojure
(ns app.billing.model
  (:require [libs.stripe-billing.model :as stripe]
            [recife.core :as r]))

;; Application responds to Stripe events emitted into model state
(r/defproc activate-subscription
  (fn [{:keys [::stripe/events] :as db}]
    (if-let [invoice (first (filter #(= :payment-succeeded (:type %)) events))]
      (assoc-in db [::subscriptions (:subscription-id invoice) :status] :active)
      db)))
```

**Option 2: Create a separate library model**

If no standard spec exists but the integration is generic:
```clojure
;; greenhouse_ats/model.clj (library model)
;; Defines: greenhouse inbound events, normalization, sync semantics.

;; interview_scheduling/model.clj (application model)
(ns interview-scheduling.model
  (:require [greenhouse-ats.model :as greenhouse]
            [recife.core :as r]))

(r/defproc import-candidate
  (fn [{:keys [::greenhouse/events] :as db}]
    (if-let [event (first (filter #(= :candidate-created (:type %)) events))]
      (assoc-in db [::candidacies (keyword (:candidate-id event))]
                {:source :greenhouse})
      db)))
```

**Option 3: Abstract and move on**

If the integration is minor, just abstract it:
```clojure
;; Do not model Slack API details, just model the domain event:
(update db :notifications/outbox conj {:to interviewer-ids :channel :slack})
```

### Red flags: integration logic in your spec

If you find yourself writing spec like this, stop and reconsider:

```clojure
;; TOO DETAILED - this is Stripe's domain, not yours
(r/defproc process-stripe-webhook
  (fn [{:keys [payload signature] :as db}]
    (when (verify-stripe-signature payload signature)
      (let [event (parse-stripe-event payload)]
        (if (= "invoice.paid" (:type event))
          ...
          db)))))
```

Instead:
```clojure
;; Application responds to normalized payment events (integration handled elsewhere)
(r/defproc payment-received
  (fn [{:keys [::billing/events] :as db}]
    (if-let [invoice (first (filter #(= :invoice-paid (:type %)) events))]
      ...
      db)))
```

### Common library model extractions

| Code pattern found | Library model candidate |
|-------------------|-------------------------|
| OAuth token exchange, refresh, session management | `oauth2/model.clj` |
| Stripe webhook handling, subscription lifecycle | `stripe_billing/model.clj` |
| Email sending with templates, bounce handling | `email_delivery/model.clj` |
| Calendar event sync, availability checking | `calendar_integration/model.clj` |
| ATS candidate import, status sync | `greenhouse_ats/model.clj`, `lever_ats/model.clj` |
| File upload, virus scanning, thumbnail generation | `file_storage/model.clj` |

See patterns.md Pattern 8 for detailed examples of integrating library models.

## Common distillation challenges

### Challenge: Duplicate terminology

When you find two terms for the same concept (across specs, within a spec, or between spec and code) treat it as a blocking problem.

```
-- BAD: Acknowledges duplication without resolving it
-- Order vs Purchase
;; checkout/model.clj uses \"Purchase\" - these are equivalent concepts.
```

This is not a resolution. When different parts of a codebase are built against different specs, both terms end up in the implementation: duplicate models, redundant join tables, foreign keys pointing both ways.

**What to do:**
- Choose one term. Cross-reference related specs before deciding.
- Update all references. Do not leave the old term in comments or "see also" notes.
- Note the rename in a changelog, not in the spec itself.

**Warning signs in code:**
- Two models representing the same concept (`Order` and `Purchase`)
- Join tables for both (`order_items`, `purchase_items`)
- Comments like "equivalent to X" or "same as Y"

The spec you extract must pick one term. Flag the other as technical debt to remove.

### Challenge: Implicit state machines

Code often has implicit states that are not modelled:

```python
# No explicit status field, but there's a state machine hiding here
class FeedbackRequest:
    interview_id = Column(Integer)
    interviewer_id = Column(Integer)
    requested_at = Column(DateTime)
    reminded_at = Column(DateTime, nullable=True)
    feedback_id = Column(Integer, nullable=True)  # FK to Feedback if submitted
```

The implicit states are:
- `pending`: requested_at set, feedback_id null, reminded_at null
- `reminded`: reminded_at set, feedback_id null
- `submitted`: feedback_id set

Extract to explicit:
```clojure
(def global
  {::feedback-requests
   {:fr-1 {:interview-id :int-1
           :interviewer-id :i-1
           :requested-at 1739181600000
           :reminded-at nil
           :status :pending}}})
```

### Challenge: Scattered logic

The same conceptual rule might be spread across multiple places:

```python
# In API handler
def accept_invitation(request):
    if invitation.status != 'pending':
        return error(400, "Already responded")
    ...

# In model
class Invitation:
    def can_accept(self):
        return self.expires_at > datetime.utcnow()

# In service
def process_acceptance(invitation, slot):
    if slot not in invitation.slots:
        raise InvalidSlot()
    ...
```

Consolidate into one rule:
```clojure
(r/defproc candidate-accepts
  (fn [{:keys [::invitations :commands/candidate-accepts :clock/now] :as db}]
    (when-let [{:keys [invitation-id slot-id]} (first candidate-accepts)]
      (let [invitation (get invitations invitation-id)]
        (when (and (= :pending (:status invitation))
                   (> (:expires-at invitation) now)
                   (contains? (set (:slot-ids invitation)) slot-id))
          ...)))))
```

### Challenge: Dead code and historical accidents

Codebases accumulate features that were built but never used, workarounds for bugs that are now fixed, and code paths that are never executed.

Do not include these in the spec. If you are unsure:
1. Check if the code is actually reachable
2. Ask developers if it is intentional
3. Check git history for context

### Challenge: Missing error handling

Code might silently fail or have incomplete error handling:

```python
def send_notification(user, message):
    try:
        slack.send(user.slack_id, message)
    except SlackError:
        pass  # Silently ignore failures
```

The spec should capture the intended behaviour, not the bug:
```clojure
(update db :notifications/outbox conj {:to user-id :channel :slack :message message})
```

Whether the current implementation properly handles failures is separate from what the system should do.

### Challenge: Over-engineered abstractions

Enterprise codebases often have abstraction layers that obscure intent:

```java
public interface NotificationStrategy {
    void notify(NotificationContext context);
}

public class SlackNotificationStrategy implements NotificationStrategy {
    @Override
    public void notify(NotificationContext context) {
        // Actual Slack call buried 5 levels deep
    }
}
```

Cut through to the actual behaviour. The model does not need strategy patterns, dependency injection or abstract factories. Just model the emitted event/state update, e.g. `(update db :notifications/outbox conj {:channel :slack ...})`.

## Checklist: Have you abstracted enough?

Before finalising a distilled model:

- [ ] No database column types (Integer, VARCHAR, etc.)
- [ ] No ORM or query syntax
- [ ] No HTTP status codes or API paths
- [ ] No framework-specific concepts (middleware, decorators, etc.)
- [ ] No programming language types (int, str, List, etc.)
- [ ] No variable names from the code (use domain terms)
- [ ] No infrastructure (Redis, Kafka, S3, etc.)
- [ ] Foreign keys replaced with relationships
- [ ] Tokens/secrets removed (implementation of identity)
- [ ] Time handling uses explicit model fields/config, not ad-hoc timedelta literals scattered through code

If any remain, ask: "Would a stakeholder include this in a requirements doc?"

## Checklist: Terminology consistency

- [ ] Each concept has exactly one name throughout the model
- [ ] No "also known as" or "equivalent to" comments
- [ ] Cross-referenced related models for conflicting terms
- [ ] Duplicate models in code flagged as technical debt to remove

## References

- [Language reference](../../references/language-reference.md) — full Recife syntax
- [Worked examples](./references/worked-examples.md) — complete code-to-spec examples in Python, TypeScript and Java
