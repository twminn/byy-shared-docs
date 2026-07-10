# BYY Shared Documentation

Shared documentation between BYY projects (Rails app, landing pages, etc.)

## Purpose

This repository serves as the single source of truth for documentation that needs to be shared between multiple BYY development teams:

- **Rails Team** — Responsible for the new Best Year Yet application built on Rails
- **Analytics Team** — Manages marketing, analytics, and reporting for Best Year Yet
- **Front End Team** — Manages the main website bestyearyet.com, campaign landing pages, and other front end customer-facing pages
- **Legacy Team** — Maintains the original Best Year Yet PRO ColdFusion application

## 🚨 Action Items

| Priority | Item | Team | Status |
|----------|------|------|--------|
| **CRITICAL** | [Tour + Signup Instrumentation](analytics/07.10.26-RAILS_TOUR_AND_SIGNUP_INSTRUMENTATION.md) | Rails | 🔲 Action Required — same-day |
| **HIGH** | [Post-Checkout Subscriber Flow](api/POST_CHECKOUT_SUBSCRIBER_FLOW.md) | Rails | 🔲 Validate user path after Stripe redirect |
| **HIGH** | [Cross-Domain Tracking — CTA Changes](analytics/CROSS_DOMAIN_TRACKING.md) | Frontend | 🔲 Action Required |
| **HIGH** | [Meta Pixel Setup in GTM](analytics/META_PIXEL_GTM_SETUP.md) | Frontend | 🔲 Pending |
| **HIGH** | [Deploy Annotation Setup — Rails](analytics/ANNOTATION_SETUP_RAILS.md) | Rails | 🔲 Action Required |
| **HIGH** | [Deploy Annotation Setup — Frontend](analytics/ANNOTATION_SETUP_FRONTEND.md) | Frontend | 🔲 Action Required |
| **HIGH** | [Deploy Annotation Setup — Legacy](analytics/ANNOTATION_SETUP_LEGACY.md) | Legacy | 🔲 Action Required |
| **MEDIUM** | [Opportunity Creation](api/OPPORTUNITY_CREATION_REQUEST.md) | Rails | ⏳ Requested |

## ✅ Recently Completed

| Item | Team | Completed |
|------|------|-----------|
| [Post-Checkout Redirect Fix](analytics/POST_CHECKOUT_REDIRECT_INCIDENT.md) | Frontend | May 6, 2026 |
| [GTM Installation on Rails](analytics/GOOGLE_ANALYTICS_GTM.md#rails-application-installation) | Rails | Dec 16, 2025 |
| [GTM on Landing Pages](analytics/GOOGLE_ANALYTICS_GTM.md) | Frontend | Dec 16, 2025 |
| [CORS Configuration](api/CORS_CONFIGURATION_REQUIRED.md) | Rails | Dec 11, 2025 |

## Structure

```
byy-shared-docs/
├── analytics/                    # Analytics & tracking (Analytics Team)
│   ├── GOOGLE_ANALYTICS_GTM.md           # GTM/GA4 setup ✅
│   ├── 07.10.26-RAILS_TOUR_AND_SIGNUP_INSTRUMENTATION.md  # 🔲 Critical Rails handoff
│   ├── CROSS_DOMAIN_TRACKING.md          # Cross-domain linking CTA changes (🔲 action required)
│   ├── ENGAGEMENT_TRACKING.md            # Engagement tracker installation for landing pages
│   ├── META_PIXEL_GTM_SETUP.md           # Meta Pixel in GTM (🔲 action required)
│   ├── ANNOTATION_SETUP_RAILS.md         # Deploy annotation setup for Rails team
│   ├── ANNOTATION_SETUP_FRONTEND.md      # Deploy annotation setup for Frontend team
│   └── ANNOTATION_SETUP_LEGACY.md        # Deploy annotation setup for Legacy team
├── api/                          # API specifications (Rails Team)
│   ├── RAILS_API_ENDPOINT_SPEC.md        # Landing page lead capture API
│   └── CORS_CONFIGURATION_REQUIRED.md    # ✅ CORS enabled
├── ghl/                          # GoHighLevel integration docs
│   ├── GOHIGHLEVEL_INTEGRATION.md        # Integration overview
│   ├── GOHIGHLEVEL_INTEGRATION_GUIDE.md  # Detailed implementation guide
│   └── GHL_PRIVATE_INTEGRATION_MIGRATION.md  # API v2 migration details
├── legacy/                       # Legacy Team (BYY PRO ColdFusion app)
│   └── README.md
└── README.md
```

## Quick Links

### For Frontend/Landing Page Team

| Document | Description |
|----------|-------------|
| [**Post-Checkout Redirect Incident**](analytics/POST_CHECKOUT_REDIRECT_INCIDENT.md) | Incident report + prevention measures for checkout dead-end |
| [**Cross-Domain Tracking**](analytics/CROSS_DOMAIN_TRACKING.md) | 🔲 **Action Required** - Update CTA buttons for purchase attribution |
| [**Meta Pixel GTM Setup**](analytics/META_PIXEL_GTM_SETUP.md) | 🔲 **Action Required** - Configure Meta Pixel in GTM |
| [**Deploy Annotation Setup**](analytics/ANNOTATION_SETUP_FRONTEND.md) | 🔲 **Action Required** - Add deploy annotations to your CI pipeline |
| [**Engagement Tracking**](analytics/ENGAGEMENT_TRACKING.md) | Install `byy-tracker.js` for scroll, CTA, form, video, and engagement tracking |
| [Landing Leads API](api/RAILS_API_ENDPOINT_SPEC.md) | API spec for `POST /api/v1/landing_leads` |
| [GHL Integration Guide](ghl/GOHIGHLEVEL_INTEGRATION_GUIDE.md) | How to integrate with GHL |
| [GTM/GA4 Setup](analytics/GOOGLE_ANALYTICS_GTM.md) | Analytics tracking implementation |

### For Rails Team

| Document | Description |
|----------|-------------|
| [**Tour + Signup Instrumentation**](analytics/07.10.26-RAILS_TOUR_AND_SIGNUP_INSTRUMENTATION.md) | 🔲 **Critical** — fire `sign_up` + free tour funnel events (same-day) |
| [**Post-Checkout Subscriber Flow**](api/POST_CHECKOUT_SUBSCRIBER_FLOW.md) | 🔲 **Action Required** - Handle Stripe redirect, create accounts, start onboarding |
| [**Deploy Annotation Setup**](analytics/ANNOTATION_SETUP_RAILS.md) | 🔲 **Action Required** - Add deploy annotations to your CI pipeline |
| [GTM Installation](analytics/GOOGLE_ANALYTICS_GTM.md#rails-application-installation) | ✅ Completed - GTM installed on Rails app |
| [CORS Configuration](api/CORS_CONFIGURATION_REQUIRED.md) | ✅ Completed - CORS enabled for landing pages |
| [API Spec](api/RAILS_API_ENDPOINT_SPEC.md) | Implementation status and notes |
| [GHL Migration](ghl/GHL_PRIVATE_INTEGRATION_MIGRATION.md) | API v2 migration details |

### For Legacy Team

| Document | Description |
|----------|-------------|
| [**Deploy Annotation Setup**](analytics/ANNOTATION_SETUP_LEGACY.md) | 🔲 **Action Required** - Add deploy annotations to your CI pipeline |
| [Legacy Folder](legacy/) | Files and tasks for the Legacy Team |

## Endpoints

### Landing Page Lead Capture

| Environment | URL |
|-------------|-----|
| **Staging** | `https://byy-staging.bestyearyet.io/api/v1/landing_leads` |
| **Production** | `https://bestyearyet.io/api/v1/landing_leads` |

## Contributing

### Updating Documentation

1. Create a branch for your changes
2. Make updates to the relevant `.md` files
3. Open a Pull Request
4. Request review from the other team if the change affects them
5. Merge after approval

### Adding New Landing Pages

When creating a new landing page that uses the lead capture API:

1. Update `api/RAILS_API_ENDPOINT_SPEC.md` with the new `page` parameter value
2. Coordinate with Rails team if new tags or fields are needed
3. Test on staging before production

### Updating GHL Configuration

When GHL configuration changes:

1. Update relevant docs in `ghl/` directory
2. Notify both teams via PR comments
3. Ensure environment variables are updated in both projects if needed

## Team Contacts

- **Rails Team**: Updates to API implementation, GHL service changes, new app features
- **Analytics Team**: Marketing analytics, tracking implementation, reporting
- **Front End Team**: Main website (bestyearyet.com), landing pages, customer-facing pages
- **Legacy Team**: BYY PRO application (ColdFusion), subscriber management, legacy integrations

## Analytics Configuration

All BYY properties use the same Google Tag Manager container for consistent analytics:

| Property | Value |
|----------|-------|
| **GTM Container ID** | `GTM-K9PVK9ZF` |
| **GA4 Measurement ID** | `G-6GPCZV3DHR` |

See [analytics/GOOGLE_ANALYTICS_GTM.md](analytics/GOOGLE_ANALYTICS_GTM.md) for full setup instructions.

## Version History

| Date | Change | Team |
|------|--------|------|
| 2026-05-07 | Added deploy annotation setup guides for Rails, Frontend, and Legacy teams | Analytics |
| 2026-05-06 | Post-checkout redirect fix + subscriber flow doc for Rails team | Frontend |
| 2026-04-21 | Added cross-domain tracking CTA changes for LP team | Analytics |
| 2025-12-16 | Added Meta Pixel GTM setup documentation | Rails |
| 2025-12-16 | GTM installed on Rails app (release-v3.0.54) | Rails |
| 2025-12-16 | Added GTM/GA4 documentation, landing pages installed | Frontend |
| 2025-12-11 | Requested opportunity creation feature | Frontend |
| 2025-12-11 | CORS configuration implemented ✅ | Rails |
| 2025-12-11 | Added CORS configuration requirements | Frontend |
| 2025-12-10 | Initial shared docs setup | Rails |
| 2025-12-10 | Added landing leads API spec | Rails |









