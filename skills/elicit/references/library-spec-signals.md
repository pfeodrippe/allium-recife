# Recognising library spec opportunities

During elicitation, watch for behavior that should live in reusable Quint modules rather than inline in an application model.

The same signals apply in distillation.

## Strong signals

### External integration language appears

- "Users sign in with Google/Microsoft/GitHub"
- "Payments are handled by Stripe/PayPal"
- "Calendar syncs with Google Calendar/Outlook"
- "Files go to S3/GCS"
- "Webhook events drive updates"

### Generic protocol behavior appears

- OAuth code exchange, token refresh, session expiry
- Subscription invoicing, retries, dunning
- Email delivery + bounce handling
- Webhook signature validation + retry policy

### Vendor names dominate requirements

If requirements are mostly provider protocol details, the behavior likely belongs in a library module (`oauth2.qnt`, `billing.qnt`, etc.).

## Questions to ask

1. Is this behavior specific to your product, or generic to any integration with this provider?
2. Would another team integrating the same provider need nearly the same model?
3. Are you describing provider mechanics, or product policy on top of provider events?
4. Should we import an existing module or create a reusable local module?

## Decision outcomes

### Use existing module

"This sounds like standard OAuth. Let’s import an OAuth module and keep this model focused on user/account policy."

### Create new reusable module

"This Greenhouse ATS flow looks reusable across hiring systems. Let’s capture it as `greenhouse-ats.qnt` and import it."

### Keep inline (rare)

If behavior is deeply product-specific and unlikely to recur, keep it inline.

## Common library module candidates

| Domain | Typical module names |
|--------|----------------------|
| Authentication | `oauth2.qnt`, `saml.qnt`, `magic-link.qnt` |
| Payments | `stripe-billing.qnt`, `subscriptions.qnt` |
| Notifications | `email-delivery.qnt`, `push.qnt` |
| Storage | `object-storage.qnt`, `file-processing.qnt` |
| Calendar | `calendar-sync.qnt` |
| Integrations | `webhook-runtime.qnt`, `rate-limiter.qnt` |

## Boundary rule

Library module owns provider mechanics.
Application module owns business consequences.

Example split:

```quint
// oauth2.qnt owns:
// - provider auth/session mechanics
// - emits auth/session lifecycle events

// app-auth.qnt owns:
// - create/update local user profile on login
// - block suspended users
// - role assignment and audit policy
```

## Red flags you missed a library opportunity

- Repeated provider protocol steps embedded across multiple features
- Vendor event names hardcoded throughout business models
- Same retry/timeout logic copied in multiple modules
