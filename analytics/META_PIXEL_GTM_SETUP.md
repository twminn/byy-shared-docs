# Meta Pixel Setup in Google Tag Manager

This guide explains how to configure Meta (Facebook) Pixel within GTM to properly track conversions, including the `Purchase` event with correct currency formatting.

## 🔑 Quick Reference

| Property | Value |
|----------|-------|
| **GTM Container ID** | `GTM-K9PVK9ZF` |
| **Meta Pixel ID** | `[YOUR_PIXEL_ID]` ← Replace with your actual Pixel ID |
| **GTM Dashboard** | [tagmanager.google.com](https://tagmanager.google.com) |
| **Meta Events Manager** | [business.facebook.com/events_manager](https://business.facebook.com/events_manager) |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Google Tag Manager (GTM-K9PVK9ZF)               │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  GA4 Tag     │  │  Meta Pixel  │  │  Other Tags  │              │
│  │  (G-6GPC...) │  │  Base + Events│  │  (future)    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│         ▲                 ▲                                         │
│         │                 │                                         │
│  ┌──────────────────────────────────────────────────┐              │
│  │              dataLayer Variables                  │              │
│  │  • event, value, currency, transaction_id, etc.  │              │
│  └──────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ dataLayer.push()
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                     BYY Applications                                 │
│                                                                      │
│  BYYAnalytics.trackPurchase(transactionId, value, plan)             │
│  BYYAnalytics.trackSignup(userId, method)                           │
│  BYYAnalytics.trackLogin(userId, method)                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Create DataLayer Variables in GTM

Before creating tags, you need to define variables that read from the dataLayer.

### 1.1 Navigate to Variables

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Select **GTM-K9PVK9ZF** container
3. Click **Variables** in the left sidebar
4. Click **New** under "User-Defined Variables"

### 1.2 Create Required Variables

Create these Data Layer Variables:

| Variable Name | Data Layer Variable Name | Type |
|--------------|--------------------------|------|
| `DLV - value` | `value` | Data Layer Variable |
| `DLV - currency` | `currency` | Data Layer Variable |
| `DLV - transaction_id` | `transaction_id` | Data Layer Variable |
| `DLV - event_id` | `event_id` | Data Layer Variable |
| `DLV - user_id` | `user_id` | Data Layer Variable |
| `DLV - method` | `method` | Data Layer Variable |
| `DLV - content_name` | `content_name` | Data Layer Variable |
| `DLV - items` | `items` | Data Layer Variable |

**For each variable:**
1. Click **New**
2. Name it (e.g., `DLV - value`)
3. Click **Variable Configuration**
4. Select **Data Layer Variable**
5. Enter the Data Layer Variable Name (e.g., `value`)
6. Click **Save**

---

## Step 2: Create Meta Pixel Base Tag

This tag loads the Meta Pixel on every page.

### 2.1 Create the Tag

1. Go to **Tags** → **New**
2. Name: `Meta Pixel - Base`
3. Click **Tag Configuration**
4. Select **Custom HTML**
5. Paste this code:

```html
<!-- Meta Pixel Code -->
<script>
!function(f,b,e,v,n,t,s)
{if(f.fbq)return;n=f.fbq=function(){n.callMethod?
n.callMethod.apply(n,arguments):n.queue.push(arguments)};
if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
n.queue=[];t=b.createElement(e);t.async=!0;
t.src=v;s=b.getElementsByTagName(e)[0];
s.parentNode.insertBefore(t,s)}(window, document,'script',
'https://connect.facebook.net/en_US/fbevents.js');
fbq('init', 'YOUR_PIXEL_ID_HERE');
fbq('track', 'PageView');
</script>
<!-- End Meta Pixel Code -->
```

> ⚠️ **Replace `YOUR_PIXEL_ID_HERE`** with your actual Meta Pixel ID

### 2.2 Set the Trigger

1. Click **Triggering**
2. Select **All Pages**
3. Click **Save**

---

## Step 3: Create Meta Pixel Purchase Event Tag

This is the critical tag that fixes the currency error.

### 3.1 Create the Tag

1. Go to **Tags** → **New**
2. Name: `Meta Pixel - Purchase`
3. Click **Tag Configuration**
4. Select **Custom HTML**
5. Paste this code:

```html
<script>
  fbq('track', 'Purchase', {
    value: {{DLV - value}},
    currency: {{DLV - currency}} || 'USD',
    content_type: 'product',
    content_ids: [{{DLV - transaction_id}}]
  }, { eventID: {{DLV - event_id}} });
</script>
```

> ⚠️ **Important:** The `eventID` parameter is required for deduplication with Conversion API (CAPI). See [Deduplication](#deduplication-with-conversion-api) section below.

### 3.2 Create the Trigger

1. Click **Triggering**
2. Click the **+** icon to create a new trigger
3. Name: `Event - purchase`
4. Click **Trigger Configuration**
5. Select **Custom Event**
6. Event name: `purchase`
7. Click **Save**

### 3.3 Set Tag Sequencing (Important!)

1. In the tag configuration, expand **Advanced Settings**
2. Expand **Tag Sequencing**
3. Check **"Fire a tag before Meta Pixel - Purchase fires"**
4. Select **Meta Pixel - Base**
5. Click **Save**

---

## Step 4: Create Additional Event Tags

### 4.1 Meta Pixel - Sign Up (CompleteRegistration)

```html
<script>
  fbq('track', 'CompleteRegistration', {
    content_name: {{DLV - method}} || 'email',
    status: 'complete'
  });
</script>
```

**Trigger:** Custom Event → `sign_up`

### 4.2 Meta Pixel - Lead

```html
<script>
  fbq('track', 'Lead', {
    content_name: 'BYY Lead',
    content_category: 'subscription'
  });
</script>
```

**Trigger:** Custom Event → `form_submit` (or specific form events)

### 4.3 Meta Pixel - InitiateCheckout

```html
<script>
  fbq('track', 'InitiateCheckout', {
    content_type: 'product',
    content_name: {{DLV - content_name}},
    value: {{DLV - value}},
    currency: {{DLV - currency}} || 'USD'
  }, { eventID: {{DLV - event_id}} });
</script>
```

**Trigger:** Custom Event → `initiate_checkout`

---

## Step 5: Testing & Verification

### 5.1 Use GTM Preview Mode

1. In GTM, click **Preview** button (top right)
2. Enter your staging URL: `https://byy-staging.bestyearyet.io`
3. Browse the site and perform actions
4. The GTM debug panel shows which tags fired

### 5.2 Use Meta Pixel Helper Chrome Extension

1. Install [Meta Pixel Helper](https://chrome.google.com/webstore/detail/meta-pixel-helper/fdgfkebogiimcoedlicjlajpkdmockpc)
2. Visit your site
3. Click the extension icon to see Pixel events
4. Verify Purchase events include `currency: 'USD'`

### 5.3 Check Meta Events Manager

1. Go to [Meta Events Manager](https://business.facebook.com/events_manager)
2. Select your Pixel
3. Click **Test Events**
4. Enter your staging URL
5. Perform a test purchase
6. Verify the Purchase event appears with correct parameters

### 5.4 Browser Console Verification

```javascript
// Check if Meta Pixel is loaded
if (typeof fbq === 'function') {
    console.log('✅ Meta Pixel is loaded');
} else {
    console.log('❌ Meta Pixel is NOT loaded');
}

// Manually test a purchase event (for debugging)
fbq('track', 'Purchase', { value: 1.00, currency: 'USD' });
```

---

## Step 6: Publish GTM Container

⚠️ **Only after testing is complete:**

1. Click **Submit** (top right)
2. Add version name: `v1.X - Add Meta Pixel with Purchase tracking`
3. Add description of changes
4. Click **Publish**

---

## Event Mapping Reference

| BYYAnalytics Function | dataLayer Event | Meta Pixel Event | Has event_id |
|-----------------------|-----------------|------------------|--------------|
| `trackSignup()` | `sign_up` | `CompleteRegistration` | No |
| `trackLogin()` | `login` | `Login` (custom) | No |
| `trackPurchase()` | `purchase` | `Purchase` | ✅ Yes |
| `trackInitiateCheckout()` | `initiate_checkout` | `InitiateCheckout` | ✅ Yes |
| `trackFeatureUsed()` | `feature_used` | `CustomEvent` | No |
| `trackFormSubmit()` | `form_submit` | `Lead` | No |

---

## Deduplication with Conversion API

If you're using both Meta Pixel (browser-side) and Conversion API (server-side), you **must** include `eventID` to prevent duplicate event counting.

### How It Works

1. **BYYAnalytics** generates a unique `event_id` (e.g., `evt_1702745123456_a3b8c9d2f`)
2. This ID is pushed to the dataLayer with the event
3. GTM passes it to Meta Pixel via the `eventID` parameter
4. The same `event_id` should be sent via CAPI
5. Meta automatically deduplicates matching events

### Event ID Format

```
evt_[timestamp]_[random_string]
Example: evt_1702745123456_a3b8c9d2f
```

### Verifying Deduplication

1. Go to [Meta Events Manager](https://business.facebook.com/events_manager)
2. Select your Pixel → **Overview**
3. Look for the **Event Deduplication** section
4. Check the duplicate rate percentage (lower is better)

---

## Troubleshooting

### "Currency parameter is missing or invalid"

**Cause:** The Purchase event is firing without a valid currency value.

**Fix:**
1. Ensure `DLV - currency` variable exists in GTM
2. Add fallback in the tag: `currency: {{DLV - currency}} || 'USD'`
3. Verify `BYYAnalytics.trackPurchase()` is being called with all parameters

### Events not appearing in Meta Events Manager

**Causes:**
1. GTM container not published
2. Meta Pixel Base tag not firing before event tags
3. Ad blocker preventing Pixel from loading

**Fix:**
1. Publish GTM container
2. Set up tag sequencing (Base tag fires first)
3. Test in incognito mode without ad blocker

### Duplicate PageView events

**Cause:** Meta Pixel base code included both in GTM and directly in page HTML.

**Fix:** Remove any direct Meta Pixel code from your HTML if using GTM.

---

## Content Security Policy (CSP)

The Rails app CSP already allows Meta/Facebook domains. If you see CSP errors, ensure these are allowed:

```ruby
# config/initializers/content_security_policy.rb
policy.script_src :self, 'https://connect.facebook.net'
policy.img_src :self, 'https://www.facebook.com'
policy.connect_src :self, 'https://www.facebook.com'
```

---

## Related Documentation

- [GTM/GA4 Setup](./GOOGLE_ANALYTICS_GTM.md) - Main GTM configuration
- [Meta Pixel Documentation](https://developers.facebook.com/docs/meta-pixel)
- [GTM Custom HTML Tags](https://support.google.com/tagmanager/answer/6107167)

---

## Changelog

| Date | Change | Team |
|------|--------|------|
| 2025-12-16 | Initial Meta Pixel GTM setup documentation | Rails |
| 2025-12-16 | Added event_id for Pixel/CAPI deduplication | Rails |
| 2025-12-16 | Added trackInitiateCheckout() with event_id | Rails |
| 2025-12-16 | Added deduplication verification section | Rails |

