# Post-Checkout Redirect — Incident Report & Prevention

**Date:** May 6, 2026
**Status:** Resolved
**Severity:** High — paid subscribers were hitting a dead end after payment
**Affected property:** `new.bestyearyet.com` (paid acquisition landing page)

## What Broke

After completing payment on `new.bestyearyet.com`, users were shown Stripe's default hosted confirmation page and never progressed to the Rails app. No account was created, no confirmation email was sent, and users could not access the plan creation workshop they just paid for.

## Root Cause

When the pricing A/B experiment (`pricing-v1`) was introduced on May 1, 2026:

1. **Before (working):** CTAs linked to `bestyearyet.io/checkout?price_id=X&source=landing_new`. The Rails app created a Stripe Checkout Session with a `success_url` that pointed back to the Rails onboarding flow.

2. **After (broken):** CTAs were changed to direct Stripe Payment Links on `pay.bestyearyet.io/b/...`. These Payment Links were created with `after_completion.type = "hosted_confirmation"` (Stripe's default), which shows a generic "payment successful" page with no redirect.

The key oversight: **when replacing the Rails-managed checkout with direct Payment Links, no one configured the Payment Links' `after_completion` setting to redirect back to the Rails app.**

## Timeline

| Date | Event |
|------|-------|
| 2026-04-06 | `bestyearyet.io/checkout` route introduced (working flow) |
| 2026-05-01 | Pricing experiment launched — 6 Payment Links created with default `hosted_confirmation` |
| 2026-05-05 | Hero CTAs also switched to Payment Links (commit `afb905f`) |
| 2026-05-06 | Issue identified via manual test (Meta Ad → checkout → dead end) |
| 2026-05-06 | Fix applied: all 6 Payment Links updated to `after_completion.type = "redirect"` |

## What Was Fixed

1. **Stripe config:** All 6 Payment Links updated via API to redirect to `https://bestyearyet.io/signup?session_id={CHECKOUT_SESSION_ID}&source=landing_new`
2. **Attribution:** Added `client_reference_id` decoration to Payment Link URLs so the Rails app can read source + UTM data from the Checkout Session
3. **CI guard:** Added `scripts/validate-checkout-flow.js` to the deploy workflow to prevent similar regressions

## Impact Assessment

- **Duration:** ~5 days (May 1–6, 2026)
- **Affected users:** Any user who completed a purchase via `new.bestyearyet.com` during this period
- **Revenue impact:** Payments were collected but users received no access to the product
- **Recovery needed:** Rails team should query Stripe for Checkout Sessions with `payment_link` IDs matching the 6 links below, created between May 1–6, to identify affected users and provision their accounts retroactively

## Payment Link Reference

| Variant | Plan | Stripe ID | URL |
|---------|------|-----------|-----|
| high | monthly | `plink_1TSPGlRqUyfRcS57rWBMzJyv` | `pay.bestyearyet.io/b/cNibJ3dw546CclBfDDaZi05` |
| high | annual | `plink_1TSPGgRqUyfRcS578Vbb8R91` | `pay.bestyearyet.io/b/4gMbJ3gIhcD8clB4YZaZi04` |
| mid | monthly | `plink_1TSPGnRqUyfRcS57V0j54PFo` | `pay.bestyearyet.io/b/eVq8wRgIhav085lajjaZi07` |
| mid | annual | `plink_1TSPGmRqUyfRcS57mx55cNzj` | `pay.bestyearyet.io/b/14A4gBcs15aG4T9dvvaZi06` |
| low | monthly | `plink_1TSPGoRqUyfRcS571N2Nhffz` | `pay.bestyearyet.io/b/14AfZj1Nnav085lbnnaZi09` |
| low | annual | `plink_1TSPGnRqUyfRcS57XPKgOPBm` | `pay.bestyearyet.io/b/aFaeVfeA9gTo99p0IJaZi08` |

## Prevention Measures

### 1. CI Validation (implemented)

`scripts/validate-checkout-flow.js` runs before every deploy and checks:
- All 6 Payment Links present and consistent between HTML and JS
- A/B variant completeness (all `data-exp-price` blocks balanced)
- `client_reference_id` attribution wiring in place

### 2. Stripe Redirect Verification Script

`scripts/verify-stripe-redirects.js` can be run with `STRIPE_SECRET_KEY` to verify all Payment Links have `after_completion.type = "redirect"` pointing to the correct URL.

### 3. Process Rules Going Forward

**Any time a checkout method is changed (new Payment Links, Checkout Session config, or redirect URLs), the following must be verified:**

- [ ] After-payment behavior sends user to the Rails app (not Stripe's hosted page)
- [ ] The redirect URL includes `session_id={CHECKOUT_SESSION_ID}` so Rails can verify payment
- [ ] The `source` param identifies which property sent the user
- [ ] Attribution data (UTM, click IDs) is passed via `client_reference_id`
- [ ] A test payment completes the full flow: payment → redirect → Rails account creation → workshop

### 4. Monitoring Recommendation

Add a GTM/GA4 event or server-side check that fires when a user arrives at `bestyearyet.io/signup` with a `session_id` param. If this event drops to zero while Payment Link checkout volume is positive, it indicates the redirect has broken again.

## Related Documents

- [Post-Checkout Flow Test Plan](../../POST_CHECKOUT_FLOW_TEST_PLAN.md)
- [Cross-Domain Tracking](CROSS_DOMAIN_TRACKING.md) — related attribution concern
- [Engagement Tracking](ENGAGEMENT_TRACKING.md) — `byy-tracker.js` CTA click events
