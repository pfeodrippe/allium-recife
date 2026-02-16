# Recognising library model opportunities

During elicitation, stay alert for descriptions that suggest a library model rather than application-specific logic. Library models are standalone Recife model modules for generic integrations that could be reused across projects.

This applies equally to distillation. When examining existing code and finding OAuth flows or payment processing, the same questions apply.

## Signals that something might be a library model

**External system integration:**

- "We use Google/Microsoft/GitHub for login"
- "Payments go through Stripe/PayPal"
- "We send emails via SendGrid/Postmark"
- "Calendar invites sync with Google Calendar"
- "We store files in S3/GCS"

**Generic patterns being described:**

- OAuth flows, session management, token refresh
- Payment processing, subscriptions, invoicing
- Email delivery, bounce handling, unsubscribes
- File upload, virus scanning, thumbnail generation
- Webhook receipt, retry logic, signature verification

**Implementation-agnostic descriptions:**

- "Users log in with their work account" (could be any SSO provider)
- "We charge them monthly" (could be any payment processor)
- "They get notified" (could be any notification infrastructure)

## Questions to ask

When you detect a potential library model, pause and explore:

1. **"Is this specific to your system, or is it a standard integration?"** If standard, it is likely a library model candidate.

2. **"Would another system integrating with [X] work the same way?"** If yes, it is definitely a library model candidate.

3. **"Do you have specific customisations to how [X] works, or is it standard?"** Standard behaviour points to a library model. Heavy customisation might still be a library model with configuration.

4. **"Should we look for an existing library model for [X], or do you need something custom?"** This encourages reuse and saves effort.

## How to handle the decision

**Option 1: Use an existing library model**

"It sounds like you're describing a standard OAuth flow. There's likely an existing library model for this. Shall we require that namespace rather than specifying OAuth details inline? Your application model would just respond to authentication events."

**Option 2: Create a new library model**

"The way you're describing this Greenhouse ATS integration sounds generic enough that it could be its own library model. Other hiring applications might integrate with Greenhouse the same way. Should we create a separate `greenhouse_ats/model.clj` module that this application model requires?"

**Option 3: Keep it inline (rare)**

"This integration is so specific to your system that it probably doesn't make sense as a standalone model. Let's include it directly."

## Common library model candidates

| Domain | Likely library models |
|--------|---------------------|
| Authentication | OAuth providers (Google, Microsoft, GitHub), SAML, magic links |
| Payments | Stripe, PayPal, subscription billing, usage-based billing |
| Communications | Email delivery, SMS, push notifications, Slack/Teams |
| Storage | S3-compatible storage, file scanning, image processing |
| Calendar | Google Calendar, Outlook, iCal feeds |
| CRM/ATS | Salesforce, HubSpot, Greenhouse, Lever |
| Analytics | Segment, Mixpanel, event tracking |
| Infrastructure | Webhook handling, rate limiting, audit logging |

## The boundary question

When you identify a library model candidate, the key question is: "Where does the library model end and the application model begin?"

The library model handles:

- The mechanics of the integration (OAuth flow, payment processing)
- Events that any consumer would care about (login succeeded, payment failed)
- Configuration that varies between deployments

The application model handles:

- What happens in your system when those events occur
- Application-specific entities (your User, your Subscription)
- Business rules unique to your domain

Example boundary:

```clojure
;; Library model (oauth/model.clj) handles:
;;   - provider configuration
;;   - token exchange
;;   - session lifecycle
;;   - appends normalized auth events to model state
;;
;; Application model handles:
;;   - creating your User projection on first login
;;   - role/permission assignment for new users
;;   - blocking suspended users from login transitions
;;   - compliance-specific audit events
```

## Red flags you missed a library model

During review, watch for:

- **Detailed protocol descriptions.** "First we redirect to Google, then they redirect back with a code, then we exchange it for a token..." This is OAuth. Use a library model.
- **Vendor-specific details.** "Stripe sends a webhook with event type `invoice.paid`..." This is Stripe integration. Use a library model.
- **Repeated patterns.** If you are specifying similar retry/timeout/error handling for multiple integrations, extract a common pattern.
