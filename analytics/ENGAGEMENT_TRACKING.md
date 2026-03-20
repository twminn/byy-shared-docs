# Landing Page Engagement Tracking

Installation guide for adding engagement tracking to the BYY landing pages (`new.bestyearyet.com`, `gift.bestyearyet.com`, and future sites).

**Owner:** Frontend Team
**Repo:** `byy-landing-pages`
**Related:** [GTM/GA4 Setup](./GOOGLE_ANALYTICS_GTM.md) | [Conversion Tracking Audit](./03.07.26-CONVERSION_TRACKING_AUDIT.md)

---

## Overview

The `byy-tracker.js` library pushes custom events to `window.dataLayer`. GTM picks them up via custom event triggers and forwards them to GA4 as tagged events with custom dimensions. The marketing analytics dashboard then queries GA4 to visualize the data.

```
byy-tracker.js → window.dataLayer → GTM triggers → GA4 events → Dashboard
```

All GA4 custom dimensions and GTM triggers/tags have already been provisioned. The only steps required from the frontend team are:

1. Load the `byy-tracker.js` script
2. Add `data-track-*` attributes to your HTML

No manual GTM configuration is needed.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **GTM Container ID** | `GTM-K9PVK9ZF` |
| **GA4 Measurement ID** | `G-6GPCZV3DHR` |
| **GA4 Property ID** | `395609610` |
| **Tracker Script** | `byy-tracker.js` (from `byy-marketing-analytics/scripts/`) |

---

## Prerequisites

Before adding engagement tracking, the page must already have GTM installed. See [GOOGLE_ANALYTICS_GTM.md](./GOOGLE_ANALYTICS_GTM.md) for installation steps. Verify GTM is working by checking that `window.google_tag_manager['GTM-K9PVK9ZF']` exists in the browser console.

---

## Step 1: Add the Tracker Script

Get `byy-tracker.js` from the `byy-marketing-analytics` repo (`scripts/byy-tracker.js`). Copy it into your project's `public/` directory.

### Option A — Script tag (recommended for landing pages)

```html
<script src="/byy-tracker.js" defer></script>
```

Place this in `<head>` (with `defer`) or just before `</body>`. The script is a self-executing IIFE — it initializes automatically on load.

### Option B — Next.js import

If your landing page is a Next.js project, copy the file to `lib/byy-tracker.js` and import it in your root layout:

```tsx
// app/layout.tsx
import '@/lib/byy-tracker';
```

Importing it once is enough — no function calls needed.

### Option C — npm package (for React apps)

For React/Next.js applications that want typed methods and module-aware tracking, use the `@marigold-one11/analytics` npm package instead. See the [Marigold Analytics ENGAGEMENT_TRACKING.md](https://github.com/twminn/marigold-analytics/blob/main/docs/ENGAGEMENT_TRACKING.md) for full npm installation instructions. The rest of this document covers the vanilla JS approach.

---

## Step 2: Add Data Attributes

The tracker auto-detects some elements, but works best with explicit `data-track-*` attributes.

### Sections

Add `data-track-section` to each major page section. The value becomes the `section_name` dimension in GA4.

```html
<section data-track-section="Hero">
  <h1>Best Year Yet</h1>
  <!-- ... -->
</section>

<section data-track-section="Features">
  <!-- ... -->
</section>

<section data-track-section="Pricing">
  <!-- ... -->
</section>

<section data-track-section="Testimonials">
  <!-- ... -->
</section>

<section data-track-section="FAQ">
  <!-- ... -->
</section>
```

The tracker fires a `section_view` event when each section becomes 50%+ visible in the viewport (via `IntersectionObserver`). Each section fires only once per page load.

### CTAs

These elements are tracked automatically — **no attributes needed**:

- `<button>` elements
- `<a class="cta">` or `<a class="btn">` links
- Elements with `role="button"`

To explicitly mark other elements as CTAs:

```html
<div data-track-cta="true" onClick="handleSignup()">Get Started Free</div>
```

The tracker captures:
- `click_text` — the element's text content (truncated to 100 chars)
- `click_url` — the `href` or `data-href` attribute
- `click_section` — the name of the parent `[data-track-section]`, if any

### Forms

Add `data-track-form` to any form you want lifecycle tracking on:

```html
<form data-track-form="signup" action="/signup" method="POST">
  <input type="email" name="email" placeholder="Enter your email" />
  <button type="submit">Sign Up</button>
</form>
```

```html
<form data-track-form="gift-checkout">
  <!-- ... -->
</form>
```

Three events are fired automatically:

| Event | When | Parameters |
|-------|------|-----------|
| `form_start` | First field receives focus | `form_name` |
| `form_submit` | Form is submitted | `form_name` |
| `form_abandon` | Page unloads with an incomplete (started but not submitted) form | `form_name`, `last_field` |

The `form_name` value comes from (in order of priority): `data-track-form` attribute, `name` attribute, `id` attribute, or `"unknown"`.

### Videos

Add `data-track-video` to `<video>` elements:

```html
<video data-track-video="Welcome Video" src="/intro.mp4" controls></video>
```

```html
<video data-track-video="Product Tour" poster="/tour-thumb.jpg" controls>
  <source src="/tour.webm" type="video/webm" />
  <source src="/tour.mp4" type="video/mp4" />
</video>
```

Events fired:

| Event | `video_action` | `video_percent` |
|-------|---------------|----------------|
| `video_engagement` | `play` | Current position % |
| `video_engagement` | `pause` | Current position % |
| `video_engagement` | `complete` | `100` |

---

## What Gets Tracked Automatically (No Attributes Needed)

These features work as soon as `byy-tracker.js` is loaded:

| Feature | Event | Details |
|---------|-------|---------|
| **Scroll depth** | `scroll_milestone` | Fires at 25%, 50%, 75%, 100% scroll depth. Each milestone fires once per page load. |
| **Engaged time** | `engaged_time` | Fires at 15s, 30s, 60s, 120s of active time on page. Pauses when the tab is hidden or the user is idle for 30+ seconds. |

---

## Full Events Reference

| Event | Parameters | Trigger |
|-------|-----------|---------|
| `scroll_milestone` | `scroll_depth` (25 / 50 / 75 / 100) | User scrolls past threshold |
| `cta_click` | `click_text`, `click_url`, `click_section` | CTA element clicked |
| `section_view` | `section_name`, `section_index` | Section 50%+ visible in viewport |
| `form_start` | `form_name` | First field interaction in a form |
| `form_abandon` | `form_name`, `last_field` | Page unload with incomplete form |
| `form_submit` | `form_name` | Form submitted |
| `video_engagement` | `video_title`, `video_percent`, `video_action` | Video play / pause / complete |
| `engaged_time` | `engaged_seconds` (15 / 30 / 60 / 120) | Active time milestone reached |

---

## GA4 Custom Dimensions

These 10 event-scoped dimensions are already registered in GA4 property `395609610`. No GA4 configuration is needed from the frontend team.

| Dimension | Type | Used By |
|-----------|------|---------|
| `scroll_depth` | Event-scoped | `scroll_milestone` |
| `click_text` | Event-scoped | `cta_click` |
| `click_url` | Event-scoped | `cta_click` |
| `click_section` | Event-scoped | `cta_click` |
| `section_name` | Event-scoped | `section_view` |
| `section_index` | Event-scoped | `section_view` |
| `form_name` | Event-scoped | `form_start`, `form_submit`, `form_abandon` |
| `video_title` | Event-scoped | `video_engagement` |
| `video_percent` | Event-scoped | `video_engagement` |
| `engaged_seconds` | Event-scoped | `engaged_time` |

---

## GTM Configuration

All triggers and tags are already configured in GTM container `GTM-K9PVK9ZF`:

- **8 Custom Event triggers** (one per event above)
- **12 Data Layer Variables** (one per parameter above, plus `last_field` and `video_action`)
- **8 GA4 Event tags** mapping each trigger to the corresponding GA4 event with parameter bindings

No GTM changes are needed from the frontend team.

---

## Example: Full Landing Page Integration

Here's a minimal example showing all tracked elements on a single page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Best Year Yet</title>

  <!-- Google Tag Manager (must be first) -->
  <script>(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
  new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
  j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
  'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
  })(window,document,'script','dataLayer','GTM-K9PVK9ZF');</script>

  <!-- Engagement Tracker -->
  <script src="/byy-tracker.js" defer></script>
</head>
<body>
  <!-- GTM noscript fallback -->
  <noscript><iframe src="https://www.googletagmanager.com/ns.html?id=GTM-K9PVK9ZF"
  height="0" width="0" style="display:none;visibility:hidden"></iframe></noscript>

  <section data-track-section="Hero">
    <h1>Make This Your Best Year Yet</h1>
    <p>Goal setting that actually works.</p>
    <button onclick="handleSignup('yearly')">Start Free Trial</button>
  </section>

  <section data-track-section="Features">
    <h2>Features</h2>
    <!-- feature content -->
  </section>

  <section data-track-section="Video">
    <video data-track-video="Product Demo" poster="/demo-thumb.jpg" controls>
      <source src="/demo.mp4" type="video/mp4" />
    </video>
  </section>

  <section data-track-section="Signup">
    <form data-track-form="email-signup">
      <input type="email" name="email" placeholder="Your email" />
      <button type="submit">Get Started</button>
    </form>
  </section>

  <section data-track-section="FAQ">
    <h2>FAQ</h2>
    <!-- FAQ content -->
  </section>
</body>
</html>
```

With just the `byy-tracker.js` script and the `data-track-*` attributes above, this page will automatically track: scroll milestones, section visibility, CTA clicks, form lifecycle, video engagement, and active time on page.

---

## Testing

### 1. GTM Preview Mode

1. Open [tagmanager.google.com](https://tagmanager.google.com) and select the BYY container (`GTM-K9PVK9ZF`)
2. Click **Preview** and enter your page URL (localhost works)
3. Interact with the page — scroll, click CTAs, focus on form fields, play videos
4. Verify each event appears in the GTM debug panel with the correct parameters

### 2. GA4 Realtime

1. Open the GA4 property (`395609610`) → **Reports → Realtime**
2. Trigger events on your page
3. Confirm events appear under "Event count by Event name"
4. Click into an event to verify custom dimensions are populated

### 3. Browser Console

Open DevTools and inspect the dataLayer directly:

```javascript
// See all pushed events
console.table(window.dataLayer);

// Filter to engagement events only
window.dataLayer.filter(e =>
  ['scroll_milestone', 'cta_click', 'section_view',
   'form_start', 'form_submit', 'video_engagement',
   'engaged_time'].includes(e.event)
);
```

---

## Checklist for New Landing Pages

When adding a new landing page:

- [ ] GTM snippet is in `<head>` and `<body>` (container `GTM-K9PVK9ZF`)
- [ ] `byy-tracker.js` is loaded (via `<script>` tag or import)
- [ ] Major page sections have `data-track-section` attributes
- [ ] Forms have `data-track-form` attributes
- [ ] Videos have `data-track-video` attributes
- [ ] CTAs use `<button>`, `<a class="cta">`, or `[data-track-cta]`
- [ ] Test with GTM Preview mode — all 8 event types fire correctly
- [ ] UTM forwarding is implemented on outbound redirects (see [Landing Page Changes](./03.07.26-LANDING_PAGE_CHANGES.md))

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| No events firing at all | GTM not loaded, or `byy-tracker.js` not loaded | Check that both scripts are in the page. Verify `window.dataLayer` exists in the console. |
| `section_view` not firing | Missing `data-track-section` attributes | Add the attribute to each `<section>` element |
| `section_view` not firing | Browser doesn't support IntersectionObserver | IE11 silently skips this feature. All modern browsers are supported. |
| `cta_click` not firing | Element is not a `<button>`, `<a class="cta/btn">`, or `[role="button"]` | Add `data-track-cta="true"` to the element |
| `form_start` not firing | Form element missing `data-track-form` | Add the attribute. The tracker only tracks forms with this attribute (or a `name`/`id`). |
| `form_abandon` unreliable | Browser limits `beforeunload` | Expected — this event is best-effort and may not fire in all browsers/scenarios |
| No scroll events | Page content shorter than viewport | Scroll events require the page to be scrollable |
| `engaged_time` milestones not firing | User is idle or tab is hidden | The timer pauses after 30s of inactivity or when the tab is backgrounded |
| Events fire in GTM but not GA4 | GA4 tag misconfigured | Check that GTM GA4 event tags use measurement ID `G-6GPCZV3DHR` |
| `click_section` is empty | CTA is not inside a `[data-track-section]` element | Wrap the CTA in a section, or accept the empty value |

---

## Notes

- The tracker is zero-dependency and works in all modern browsers (ES5+)
- Scroll milestones are deduplicated per page load
- Section visibility fires once per section per page load
- The tracker uses `MutationObserver` to pick up dynamically added elements (sections, videos)
- `form_abandon` fires on `beforeunload` — it is best-effort
- Engaged time pauses when the tab is hidden or the user is idle for 30+ seconds
- The tracker script is an IIFE — it does not pollute the global scope

---

## Changelog

| Date | Change | Author |
|------|--------|--------|
| 2026-03-16 | Initial engagement tracking installation guide | Analytics |
