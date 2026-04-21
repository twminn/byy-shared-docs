# Landing Page Cross-Domain Tracking — Required Changes

**Date:** April 21, 2026
**Status:** Action required on all landing pages
**Priority:** High — purchases are currently unattributed

## Background

When a visitor clicks a CTA on a landing page (e.g. `gift.bestyearyet.com`) and navigates to the checkout on `bestyearyet.io`, GA4 loses the session context because these are different registered domains. The original UTM source/medium is lost and the purchase shows up as `(direct) / (none)` — meaning we can't attribute revenue back to marketing campaigns.

We've configured GA4 cross-domain linking in the GTM container (version 11, published April 21 2026). This makes GA4 automatically append a `_gl` parameter to outbound links between our domains, preserving session attribution across the domain boundary.

**However, this only works automatically for standard `<a>` tag links.** Our audit of `gift.bestyearyet.com` found that the primary CTA buttons use `<button>` elements with JavaScript `window.location` navigation, which GA4's auto-linker does not intercept.

## Affected Pages

Every landing page that links to `bestyearyet.io` for checkout:

| Landing Page | Domain | CTAs Found Using `<button>` |
|---|---|---|
| Gift page | `gift.bestyearyet.com` | "Give the Annual Plan →", "Give the Monthly Plan →" |
| New page | `new.bestyearyet.com` | Needs audit — likely same pattern |
| Main site | `bestyearyet.com` | Needs audit — any pricing CTAs linking to `.io` |

Navigation links that already use `<a>` tags (e.g. "Tour the app", "Log in" in the header) are automatically handled by the GTM cross-domain config and require no changes.

## What Needs to Change

There are two options. **Option A is preferred** because it works automatically with no ongoing maintenance.

### Option A: Change CTA buttons to anchor links (preferred)

Replace `<button>` elements that navigate to `bestyearyet.io` with `<a>` tags styled as buttons.

**Before:**

```html
<button id="annualGiftBtn" class="btn btn-primary pricing-cta"
        onclick="window.location='https://bestyearyet.io/gifts?plan=yearly&source=gift_landing'">
  Give the Annual Plan →
</button>
```

**After:**

```html
<a href="https://bestyearyet.io/gifts?plan=yearly&source=gift_landing"
   class="btn btn-primary pricing-cta">
  Give the Annual Plan →
</a>
```

GA4's auto-linker will automatically append the `_gl` parameter when the user clicks the link. The resulting navigation URL will look like:

```
https://bestyearyet.io/gifts?plan=yearly&source=gift_landing&_gl=1~a1b2c3...
```

No additional JavaScript is needed.

### Option B: Use the `byyDecorateUrl` helper

If changing from `<button>` to `<a>` is not feasible (e.g. the button triggers validation or analytics before navigating), use the `byyDecorateUrl` helper that's available in `byy-tracker.js`.

**Before:**

```javascript
window.location = 'https://bestyearyet.io/gifts?plan=yearly&source=gift_landing';
```

**After:**

```javascript
window.location = window.byyDecorateUrl('https://bestyearyet.io/gifts?plan=yearly&source=gift_landing');
```

The helper creates a temporary `<a>` element, lets GA4's linker decorate it, and returns the URL with the `_gl` parameter appended. It's a synchronous call and falls back to the original URL if anything fails.

`byy-tracker.js` must be loaded on the page for this to work (it should already be included on all landing pages).

### For React/Next.js components

If CTAs are React components, use Next.js `<Link>` or a plain `<a>` tag for cross-domain navigation. Do **not** use `router.push()` for links to `bestyearyet.io` — the router's programmatic navigation won't be intercepted by GA4's linker.

```jsx
// Don't do this — _gl parameter won't be appended
<button onClick={() => router.push('https://bestyearyet.io/gifts?plan=yearly')}>
  Give the Annual Plan →
</button>

// Do this instead — GA4 auto-linker will decorate the href
<a href="https://bestyearyet.io/gifts?plan=yearly&source=gift_landing"
   className="btn btn-primary pricing-cta">
  Give the Annual Plan →
</a>
```

## How to Verify

After making changes, verify that the `_gl` parameter is being appended:

1. Open the landing page in Chrome with DevTools Network tab open
2. Click a CTA button that navigates to `bestyearyet.io`
3. Check the navigation request URL — it should contain `&_gl=1~...`
4. If using GTM Preview mode, you should see the "Linker - Decorate" event in the debug panel

You can also check by hovering over a CTA `<a>` link — the browser status bar should show the URL with `_gl=` appended (GA4 decorates on mousedown).

## What NOT to Change

- **Do not add `_gl` parameters manually.** The parameter is dynamic and session-specific. Let GA4's auto-linker handle it.
- **Do not use server-side redirects (301/302) between domains.** If any server middleware redirects from LP domains to `bestyearyet.io`, the `_gl` parameter will be stripped. Links must point directly to the destination URL.
- **Do not remove UTM parameters from CTA links.** UTMs on the initial LP visit are still important. The cross-domain linker preserves the session that carries those UTMs to checkout — they work together.

## Domains Covered by Cross-Domain Linking

The following domains are configured in GTM for cross-domain session linking:

- `bestyearyet.io` (Rails app / checkout)
- `bestyearyet.com` (main marketing site, includes `www` automatically)
- `new.bestyearyet.com` (new landing page)
- `gift.bestyearyet.com` (gift landing page)

If new landing page subdomains are added in the future, they need to be added to the GTM cross-domain config. Run:

```bash
python -m scripts.configure_cross_domain
```

to audit the current config, and add `--configure --publish` flags to update it.

## Questions

Contact the analytics team or open an issue in the `byy-marketing-analytics` repo.
