# Google Analytics & Google Tag Manager Integration

This document describes the Google Analytics 4 (GA4) and Google Tag Manager (GTM) setup for all Best Year Yet properties, including landing pages and the Rails application.

## 🔑 Quick Reference

| Property | Value |
|----------|-------|
| **GTM Container ID** | `GTM-K9PVK9ZF` |
| **GA4 Measurement ID** | `G-6GPCZV3DHR` |
| **GTM Dashboard** | [tagmanager.google.com](https://tagmanager.google.com) |
| **GA4 Dashboard** | [analytics.google.com](https://analytics.google.com) |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Google Tag Manager                          │
│                     (GTM-K9PVK9ZF)                              │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  GA4 Tag     │  │  FB Pixel    │  │  Other Tags  │          │
│  │  (G-6GPC...) │  │  (future)    │  │  (future)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ dataLayer.push()
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        Websites                                  │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ new.byy.com │  │ gift.byy.com│  │ bestyearyet │  ✅ Installed│
│  │  (Landing)  │  │  (Landing)  │  │  .com (Main)│             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌─────────────────────────────────────────────────┐            │
│  │            bestyearyet.io (Rails App)           │ 🔲 Pending │
│  └─────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

## Why GTM + GA4?

We use **GTM to manage GA4** rather than direct GA4 installation because:

1. **Single point of management** - Add/modify marketing tags without code changes
2. **Flexibility** - Easy to add Facebook Pixel, LinkedIn Insight, etc. later
3. **Version control** - GTM has built-in versioning and preview mode
4. **Cross-property consistency** - Same container works across all properties

---

## Installation Status

| Property | URL | Status |
|----------|-----|--------|
| new.bestyearyet.com | Landing page | ✅ Installed |
| gift.bestyearyet.com | Landing page | ✅ Installed |
| bestyearyet.com | Main website | ✅ Installed |
| **bestyearyet.io** | **Rails App** | **🔲 Action Required** |

---

## Rails Application Installation

### Step 1: Add GTM to Application Layout

In your main layout file (e.g., `app/views/layouts/application.html.erb`), add the GTM snippets:

**In the `<head>` section (as high as possible, after charset and viewport):**

```erb
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <!-- Google Tag Manager -->
  <script>(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
  new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
  j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
  'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
  })(window,document,'script','dataLayer','GTM-K9PVK9ZF');</script>
  <!-- End Google Tag Manager -->
  
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <!-- ... rest of head ... -->
</head>
```

**Immediately after the opening `<body>` tag:**

```erb
<body>
  <!-- Google Tag Manager (noscript) -->
  <noscript><iframe src="https://www.googletagmanager.com/ns.html?id=GTM-K9PVK9ZF"
  height="0" width="0" style="display:none;visibility:hidden"></iframe></noscript>
  <!-- End Google Tag Manager (noscript) -->
  
  <!-- ... rest of body ... -->
</body>
```

### Step 2: Create Analytics Helper (Optional but Recommended)

Create a JavaScript module for consistent event tracking:

```javascript
// app/javascript/analytics/gtm.js

const BYYAnalytics = {
    /**
     * Push event to dataLayer
     */
    pushEvent: function(eventName, params = {}) {
        window.dataLayer = window.dataLayer || [];
        dataLayer.push({ event: eventName, ...params });
        
        if (this._isDebug()) {
            console.log('[BYY Analytics]', eventName, params);
        }
    },
    
    /**
     * Track user login
     */
    trackLogin: function(userId, method = 'email') {
        this.pushEvent('login', { 
            user_id: userId, 
            method: method 
        });
    },
    
    /**
     * Track user signup
     */
    trackSignup: function(userId, method = 'email') {
        this.pushEvent('sign_up', { 
            user_id: userId, 
            method: method 
        });
    },
    
    /**
     * Track goal creation
     */
    trackGoalCreated: function(goalType, goalCategory = '') {
        this.pushEvent('goal_created', { 
            goal_type: goalType,
            goal_category: goalCategory
        });
    },
    
    /**
     * Track commitment completion
     */
    trackCommitmentCompleted: function(commitmentType) {
        this.pushEvent('commitment_completed', { 
            commitment_type: commitmentType 
        });
    },
    
    /**
     * Track subscription purchase
     */
    trackPurchase: function(transactionId, value, plan) {
        this.pushEvent('purchase', {
            transaction_id: transactionId,
            value: value,
            currency: 'USD',
            items: [{ item_name: plan, price: value }]
        });
    },
    
    /**
     * Track feature usage
     */
    trackFeatureUsed: function(featureName) {
        this.pushEvent('feature_used', { 
            feature_name: featureName 
        });
    },
    
    /**
     * Set user properties (call after login)
     */
    setUserProperties: function(userId, properties = {}) {
        this.pushEvent('user_properties_set', {
            user_id: userId,
            ...properties
        });
    },
    
    _isDebug: function() {
        return window.location.hostname === 'localhost' || 
               window.location.hostname.includes('staging');
    }
};

// Export for use in Rails
window.BYYAnalytics = BYYAnalytics;
export default BYYAnalytics;
```

### Step 3: Import in Application

In your main JavaScript entry point:

```javascript
// app/javascript/application.js
import BYYAnalytics from './analytics/gtm';

// Make available globally
window.BYYAnalytics = BYYAnalytics;
```

### Step 4: Use in Views/Controllers

**In JavaScript (Stimulus controllers, etc.):**

```javascript
// Track login
BYYAnalytics.trackLogin(userId, 'email');

// Track goal creation
BYYAnalytics.trackGoalCreated('personal', 'health');

// Track purchase
BYYAnalytics.trackPurchase('sub_123', 14.99, 'Monthly Premium');
```

**In ERB views (inline script):**

```erb
<% if flash[:signup_complete] %>
<script>
  BYYAnalytics.trackSignup('<%= current_user.id %>', 'email');
</script>
<% end %>
```

---

## Standard Events Reference

Use these event names consistently across all properties:

| Event | When to Fire | Parameters |
|-------|--------------|------------|
| `page_view` | Page load / route change | `page_path`, `page_title` |
| `login` | User logs in | `user_id`, `method` |
| `sign_up` | New user registration | `user_id`, `method` |
| `goal_created` | User creates a goal | `goal_type`, `goal_category` |
| `commitment_completed` | User completes commitment | `commitment_type` |
| `purchase` | Subscription purchased | `transaction_id`, `value`, `currency` |
| `feature_used` | Key feature interaction | `feature_name` |
| `form_submit` | Form submission | `form_name`, `form_location` |
| `button_click` | CTA button click | `button_name`, `button_location` |

---

## Server-Side Tracking (Optional)

For server-side event tracking (e.g., webhook-triggered events), use the GA4 Measurement Protocol:

```ruby
# app/services/analytics_service.rb
class AnalyticsService
  GA4_MEASUREMENT_ID = 'G-6GPCZV3DHR'
  GA4_API_SECRET = Rails.application.credentials.dig(:google, :ga4_api_secret)
  
  def self.track_event(client_id:, event_name:, params: {})
    return unless GA4_API_SECRET.present?
    
    uri = URI("https://www.google-analytics.com/mp/collect?measurement_id=#{GA4_MEASUREMENT_ID}&api_secret=#{GA4_API_SECRET}")
    
    payload = {
      client_id: client_id,
      events: [{
        name: event_name,
        params: params
      }]
    }
    
    Net::HTTP.post(uri, payload.to_json, 'Content-Type' => 'application/json')
  rescue => e
    Rails.logger.error "[Analytics] Failed to track event: #{e.message}"
  end
end
```

**Usage:**

```ruby
# Track server-side event
AnalyticsService.track_event(
  client_id: session.id.to_s,
  event_name: 'subscription_renewed',
  params: { plan: 'annual', value: 149.99 }
)
```

---

## Testing & Debugging

### GTM Preview Mode

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Select the BYY container (GTM-K9PVK9ZF)
3. Click **Preview** button
4. Enter your staging URL
5. Browse your site - events will show in the preview panel

### GA4 DebugView

1. Go to [analytics.google.com](https://analytics.google.com)
2. Navigate to Admin → DebugView
3. Events from preview sessions appear in real-time

### Browser Console

Check if GTM is loaded:

```javascript
// Verify GTM installed
if (window.google_tag_manager && window.google_tag_manager['GTM-K9PVK9ZF']) {
    console.log('✅ GTM is installed');
} else {
    console.log('❌ GTM is NOT installed');
}

// Check dataLayer
console.log('dataLayer:', window.dataLayer);
```

---

## Verification Checklist

Before deploying GTM to production:

- [ ] GTM head snippet in layout `<head>` section
- [ ] GTM noscript in layout immediately after `<body>`
- [ ] Container ID is `GTM-K9PVK9ZF`
- [ ] Test on staging with GTM Preview mode
- [ ] Key events firing correctly (login, signup, purchase)
- [ ] No JavaScript errors in console
- [ ] Events appear in GA4 DebugView

---

## Environment-Specific Notes

### Staging
- GTM Preview mode works on staging URLs
- Debug logging enabled automatically
- Safe to test event tracking

### Production
- Debug logging disabled
- All events flow to production GA4 property
- Monitor GA4 Realtime reports after deploy

---

## Troubleshooting

### GTM Not Loading

**Symptom:** No `google_tag_manager` object in console

**Causes:**
- GTM snippet missing from layout
- Content Security Policy blocking GTM
- JavaScript error before GTM loads
- Turbo/Turbolinks interfering

**Fix for Turbo/Turbolinks:**

```javascript
// Ensure dataLayer persists across Turbo navigations
document.addEventListener('turbo:load', function() {
    window.dataLayer = window.dataLayer || [];
});
```

### Events Not Appearing in GA4

**Causes:**
- GTM container not published
- GA4 tag not configured in GTM
- Wrong measurement ID

**Fix:** Verify in GTM that GA4 Configuration tag exists with correct measurement ID

---

## Related Documentation

- [Landing Page Implementation](../README.md) - How landing pages implement GTM
- [CORS Configuration](../api/CORS_CONFIGURATION_REQUIRED.md) - API access requirements
- [GTM Help Center](https://support.google.com/tagmanager)
- [GA4 Documentation](https://developers.google.com/analytics/devguides/collection/ga4)

---

## Changelog

| Date | Change | Team |
|------|--------|------|
| 2025-12-16 | Initial GTM/GA4 setup documentation | Frontend |
| 2025-12-16 | Landing pages GTM installation complete | Frontend |
| TBD | Rails app GTM installation | Rails |

