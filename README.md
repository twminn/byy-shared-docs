# BYY Shared Documentation

Shared documentation between BYY projects (Rails app, landing pages, etc.)

## Purpose

This repository serves as the single source of truth for documentation that needs to be shared between multiple BYY development teams:

- **Rails App Team** (`byy` repo)
- **Frontend/Landing Page Team** (landing page repos)

## 🚨 Action Items

| Priority | Item | Team | Status |
|----------|------|------|--------|
| **HIGH** | [Meta Pixel Setup in GTM](analytics/META_PIXEL_GTM_SETUP.md) | Frontend | 🔲 Pending |
| **MEDIUM** | [Opportunity Creation](api/OPPORTUNITY_CREATION_REQUEST.md) | Rails | ⏳ Requested |

## ✅ Recently Completed

| Item | Team | Completed |
|------|------|-----------|
| [GTM Installation on Rails](analytics/GOOGLE_ANALYTICS_GTM.md#rails-application-installation) | Rails | Dec 16, 2025 |
| [GTM on Landing Pages](analytics/GOOGLE_ANALYTICS_GTM.md) | Frontend | Dec 16, 2025 |
| [CORS Configuration](api/CORS_CONFIGURATION_REQUIRED.md) | Rails | Dec 11, 2025 |

## Structure

```
byy-shared-docs/
├── analytics/                    # Analytics & tracking
│   ├── GOOGLE_ANALYTICS_GTM.md           # GTM/GA4 setup ✅
│   ├── ENGAGEMENT_TRACKING.md            # Engagement tracker installation for landing pages
│   └── META_PIXEL_GTM_SETUP.md           # Meta Pixel in GTM (🔲 action required)
├── api/                          # API specifications
│   ├── RAILS_API_ENDPOINT_SPEC.md        # Landing page lead capture API
│   └── CORS_CONFIGURATION_REQUIRED.md    # ✅ CORS enabled
├── ghl/                          # GoHighLevel integration docs
│   ├── GOHIGHLEVEL_INTEGRATION.md        # Integration overview
│   ├── GOHIGHLEVEL_INTEGRATION_GUIDE.md  # Detailed implementation guide
│   └── GHL_PRIVATE_INTEGRATION_MIGRATION.md  # API v2 migration details
└── README.md
```

## Quick Links

### For Frontend/Landing Page Team

| Document | Description |
|----------|-------------|
| [**Meta Pixel GTM Setup**](analytics/META_PIXEL_GTM_SETUP.md) | 🔲 **Action Required** - Configure Meta Pixel in GTM |
| [**Engagement Tracking**](analytics/ENGAGEMENT_TRACKING.md) | Install `byy-tracker.js` for scroll, CTA, form, video, and engagement tracking |
| [Landing Leads API](api/RAILS_API_ENDPOINT_SPEC.md) | API spec for `POST /api/v1/landing_leads` |
| [GHL Integration Guide](ghl/GOHIGHLEVEL_INTEGRATION_GUIDE.md) | How to integrate with GHL |
| [GTM/GA4 Setup](analytics/GOOGLE_ANALYTICS_GTM.md) | Analytics tracking implementation |

### For Rails Team

| Document | Description |
|----------|-------------|
| [GTM Installation](analytics/GOOGLE_ANALYTICS_GTM.md#rails-application-installation) | ✅ Completed - GTM installed on Rails app |
| [CORS Configuration](api/CORS_CONFIGURATION_REQUIRED.md) | ✅ Completed - CORS enabled for landing pages |
| [API Spec](api/RAILS_API_ENDPOINT_SPEC.md) | Implementation status and notes |
| [GHL Migration](ghl/GHL_PRIVATE_INTEGRATION_MIGRATION.md) | API v2 migration details |

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

- **Rails Team**: Updates to API implementation, GHL service changes
- **Frontend Team**: New landing pages, API usage questions

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
| 2025-12-16 | Added Meta Pixel GTM setup documentation | Rails |
| 2025-12-16 | GTM installed on Rails app (release-v3.0.54) | Rails |
| 2025-12-16 | Added GTM/GA4 documentation, landing pages installed | Frontend |
| 2025-12-11 | Requested opportunity creation feature | Frontend |
| 2025-12-11 | CORS configuration implemented ✅ | Rails |
| 2025-12-11 | Added CORS configuration requirements | Frontend |
| 2025-12-10 | Initial shared docs setup | Rails |
| 2025-12-10 | Added landing leads API spec | Rails |









