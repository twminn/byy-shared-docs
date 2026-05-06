# Post-Checkout Subscriber Flow — Rails App Integration

**Date:** May 6, 2026
**Status:** Action required — validate subscriber path
**Priority:** High — users are now arriving at `/signup` after Stripe payment

## Overview

As of May 6, 2026, all paid subscription checkouts from `new.bestyearyet.com` redirect to the Rails app after payment. The Rails app must handle these redirects to complete the subscriber onboarding.

## Flow Diagram

```
new.bestyearyet.com           Stripe (pay.bestyearyet.io)           bestyearyet.io (Rails)
─────────────────             ──────────────────────────           ──────────────────────
User clicks CTA ──────────►  Payment Link page loads
                              User enters payment info
                              Payment succeeds
                              ◄── after_completion: redirect ──►  GET /signup?session_id={cs_xxx}&source=landing_new
                                                                  │
                                                                  ├─ Retrieve Checkout Session via Stripe API
                                                                  ├─ Extract customer email from session
                                                                  ├─ Read client_reference_id for attribution
                                                                  ├─ Create user account (or find existing)
                                                                  ├─ Attach subscription to user
                                                                  ├─ Send confirmation email
                                                                  ├─ Set password (or magic link)
                                                                  └─ Redirect to plan creation workshop
```

## Redirect Parameters

After successful payment, Stripe redirects the user's browser to:

```
https://bestyearyet.io/signup?session_id={CHECKOUT_SESSION_ID}&source=landing_new
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| `session_id` | `cs_live_...` or `cs_test_...` | Stripe Checkout Session ID — use to retrieve payment details |
| `source` | `landing_new` | Identifies the referring property (new.bestyearyet.com) |

## Checkout Session Data Available

When you retrieve the Checkout Session via `Stripe::Checkout::Session.retrieve(session_id)`:

| Field | What it contains |
|-------|-----------------|
| `customer` | Stripe Customer ID (created during checkout) |
| `customer_details.email` | User's email address |
| `customer_details.name` | User's name (if collected) |
| `subscription` | Stripe Subscription ID |
| `payment_status` | Should be `"paid"` |
| `client_reference_id` | JSON-encoded attribution: `{"source":"landing_new","page":"new","s":"facebook","m":"paid","c":"campaign_name","fb":"1"}` |
| `metadata` | Payment Link metadata (currently empty) |

### client_reference_id Format

```json
{
  "source": "landing_new",
  "page": "new",
  "s": "utm_source value",
  "m": "utm_medium value", 
  "c": "utm_campaign value",
  "fb": "1",  // present if fbclid was captured
  "gc": "1"   // present if gclid was captured
}
```

Decode with: `JSON.parse(URI.decode_www_form_component(session.client_reference_id))`

Note: This value is URL-encoded and capped at 200 characters by the front-end.

## Required Rails Implementation

### 1. Handle GET /signup with session_id

The `/signup` route (or a dedicated `/checkout/success` route) needs to:

```ruby
# Pseudocode — adapt to your controller structure
def signup
  return render_normal_signup unless params[:session_id].present?
  
  session = Stripe::Checkout::Session.retrieve(params[:session_id])
  
  # Verify payment was successful
  unless session.payment_status == 'paid'
    redirect_to signup_path, alert: "Payment not confirmed"
    return
  end
  
  # Find or create user
  email = session.customer_details&.email
  user = User.find_by(email: email) || User.new(email: email)
  
  # Attach Stripe customer + subscription
  user.stripe_customer_id = session.customer
  user.save!
  
  # Create subscription record
  subscription = Stripe::Subscription.retrieve(session.subscription)
  # ... store subscription details ...
  
  # Parse attribution for GHL
  if session.client_reference_id.present?
    attribution = JSON.parse(URI.decode_www_form_component(session.client_reference_id))
    # Store UTM source, medium, campaign for GHL pipeline tracking
  end
  
  # Trigger onboarding
  send_welcome_email(user)
  sign_in(user)  # or send password setup link
  redirect_to workshop_path  # first page of plan creation
end
```

### 2. Send Confirmation Email

After account creation, send a welcome/confirmation email that includes:
- Confirmation of their subscription (plan name, price)
- Link to set a password (if using magic link for initial auth)
- Link to begin the plan creation workshop
- Support contact info

### 3. GHL Pipeline Update

Per the existing `SignupController` flow, create/update the GHL contact and D2C pipeline opportunity:
- Contact: find-or-create by email
- Opportunity: create in `GHL_PIPELINE_D2C_YEAR1_ID` at appropriate stage
- Tags: add source attribution from `client_reference_id`

### 4. Redirect to Workshop

After account setup, redirect the user to the first page of the plan creation workshop (the "best year vision" / "first 90-day plan" flow).

## Validation Checklist

Use this checklist to confirm the flow works end-to-end:

- [ ] `GET /signup?session_id=cs_test_xxx&source=landing_new` retrieves the Checkout Session
- [ ] User account is created with correct email from the session
- [ ] Stripe subscription is linked to the user record
- [ ] Confirmation email is sent
- [ ] User can set a password (or is signed in via magic link)
- [ ] User is redirected to the plan creation workshop
- [ ] GHL contact is created/updated with D2C pipeline opportunity
- [ ] Attribution from `client_reference_id` is stored (utm_source, utm_medium, etc.)
- [ ] Existing users (same email) are handled gracefully (no duplicate accounts)
- [ ] Invalid/expired session_id shows a helpful error message

## Recovery: Affected Users (May 1–6)

During May 1–6, 2026, users who paid via these Payment Links were NOT redirected to the Rails app. Their payments were collected but no accounts were created.

**To identify affected users:**

```ruby
# Query Stripe for checkout sessions from the 6 Payment Links during the incident window
payment_link_ids = %w[
  plink_1TSPGlRqUyfRcS57rWBMzJyv
  plink_1TSPGgRqUyfRcS578Vbb8R91
  plink_1TSPGnRqUyfRcS57V0j54PFo
  plink_1TSPGmRqUyfRcS57mx55cNzj
  plink_1TSPGoRqUyfRcS571N2Nhffz
  plink_1TSPGnRqUyfRcS57XPKgOPBm
]

sessions = Stripe::Checkout::Session.list(
  payment_link: payment_link_ids,
  created: { gte: Time.new(2026, 5, 1).to_i, lte: Time.new(2026, 5, 6, 12, 0, 0).to_i }
)

sessions.auto_paging_each do |session|
  next unless session.payment_status == 'paid'
  email = session.customer_details&.email
  # Check if user exists; if not, create account and send welcome email
end
```

## Related Documents

- [Post-Checkout Redirect Incident Report](../analytics/POST_CHECKOUT_REDIRECT_INCIDENT.md) — what broke and why
- [Post-Checkout Flow Test Plan](../../POST_CHECKOUT_FLOW_TEST_PLAN.md) — full test criteria
- [Rails API Endpoint Spec](RAILS_API_ENDPOINT_SPEC.md) — existing lead capture API
- [GHL Integration Guide](../ghl/GOHIGHLEVEL_INTEGRATION_GUIDE.md) — D2C pipeline details
