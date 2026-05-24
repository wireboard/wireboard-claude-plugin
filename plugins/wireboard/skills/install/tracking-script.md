# Tracking script (`wireboard.js`)

The script that captures pageviews, sessions, and visitors. One per site.

## Snippet anatomy

The user pastes this from their WireBoard dashboard (do NOT template one with placeholder IDs). If they ask where to find it, point them to the site manager:

- **Direct link**: [wireboard.io/dashboard?action=site-edit](https://wireboard.io/dashboard?action=site-edit), select the site, click **Install**, copy the script block.
- **Manual path**: [wireboard.io/dashboard](https://wireboard.io/dashboard), click **Manage your Sites** in the sidebar, select the site, click **Install**.

```html
<!-- wireboard tag -->
<script type="text/javascript">
;(function(w,i,r,e,b,oar,d){if(!w[b]){w.WireBoardNamespace=w.WireBoardNamespace||[];
w.WireBoardNamespace.push(b);w[b]=function(){(w[b].q=w[b].q||[]).push(arguments)};
w[b].q=w[b].q||[];oar=i.createElement(r);d=i.getElementsByTagName(r)[0];oar.async=1;
oar.src=e;d.parentNode.insertBefore(oar,d)}}(window,document,"script",
"https://static.wireboard.io/wireboard.js","wireboard"));
wireboard('newTracker', 'wb', 'pipeline-0.collector.wireboard.io', {
    appId: 'AbCd1234',
    forceSecureTracker: true,
    contexts: { performanceTiming: true }
});
window.wireboard('enableActivityTracking', 5, 10);
var customContext=[{schema:'wb:io.wireboard/publisher',data:{publisher:'YOUR-PUBLISHER-UUID'}}]
window.wireboard('trackPageView', null, customContext);
</script>
```

Each line has a purpose. Paste verbatim:

| Line               | Purpose                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------ |
| First block        | Asynchronously loads `wireboard.js` from the CDN.                                                      |
| `newTracker`       | Initializes the tracker with HTTPS forcing and Performance Timing capture.                             |
| `enableActivityTracking(5, 10)` | Engagement tracking: 5-second minimum visit length, 10-second heartbeat. Powers session duration and bounce metrics. |
| `customContext`    | Carries the Publisher UUID alongside every pageview, attributing it to the user's WireBoard account.   |
| `trackPageView`    | Fires the first pageview.                                                                              |

The `appId` (8-char site UID) and Publisher UUID are user-specific. The collector hostname (e.g. `pipeline-0.collector.wireboard.io`) is per-site too. Never invent any of these.

## Where to install it

Place the snippet inside `<head>`, as early as possible (ideally first inside `<head>`). Earlier is better for Performance Timing accuracy, but the script is self-async so it does not block render.

| Framework             | Target file                                                                                       | How                                                                                                                                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Plain HTML            | every page's `<head>`, or a shared header partial                                                 | Paste the snippet verbatim.                                                                                                                                                                               |
| Next.js (App Router)  | `app/layout.tsx`, inside `<head>` of the root layout                                              | Use a raw `<script dangerouslySetInnerHTML={{__html: '...'}} />` inside `<head>`. The JS body (everything between `<script>` and `</script>` in the snippet) goes in `__html`.                            |
| Next.js (Pages Router)| `pages/_document.tsx`, inside the `<Head>` element                                                | Create `_document.tsx` if it doesn't exist. Import `Head` from `next/document` (NOT from `next/head`). Use `<script dangerouslySetInnerHTML={{__html: '...'}} />` inside `<Head>`.                        |
| Nuxt 3                | `nuxt.config.ts`, under `app.head.script`                                                         | `[{ innerHTML: '...', tagPosition: 'head' }]`. Use `innerHTML` (NOT `textContent` or `children`). The JS body goes in the string.                                                                         |
| SvelteKit             | `src/app.html`                                                                                    | Paste the snippet inside `<head>` before the `%sveltekit.head%` placeholder.                                                                                                                              |
| Remix                 | `app/root.tsx`                                                                                    | Add `<script dangerouslySetInnerHTML={{__html: '...'}} />` inside `<head>` of the root document component.                                                                                                |
| Astro                 | the site's main layout (e.g. `src/layouts/Layout.astro`)                                          | Paste inside `<head>`, BUT add the `is:inline` directive so Astro does not bundle it: `<script is:inline type="text/javascript">...</script>`.                                                            |
| Vite + React          | `index.html`                                                                                      | Paste the snippet inside `<head>` (`index.html` is a static HTML file).                                                                                                                                   |
| Vue (Vite)            | `index.html`                                                                                      | Same as Vite + React.                                                                                                                                                                                     |
| Laravel Blade         | layout view's `<head>` (typically `resources/views/layouts/app.blade.php`)                        | Paste between `<head>` and `</head>`.                                                                                                                                                                     |
| Rails                 | `app/views/layouts/application.html.erb`                                                          | Paste inside `<head>`. Do not wrap in `content_tag` helpers.                                                                                                                                              |
| Django                | `base.html` template                                                                              | Paste inside `<head>`. No `{% load static %}` needed since the script URL is absolute.                                                                                                                    |
| WordPress             | Recommend the [WireBoard Analytics plugin](https://wordpress.org/plugins/wireboard-analytics).    | Manual fallback: theme's `header.php` before `</head>`, or a "Code Snippets" / "Insert Headers and Footers" plugin. Do not edit `wp-config.php` or core files.                                            |
| Wix / Squarespace     | Built-in custom code panel, header section                                                        | Not a code change. Direct the user to paste it in the platform's dashboard.                                                                                                                               |

For framework-managed `<Script>` components (`next/script`, Nuxt `useHead`, etc.), prefer raw inline `<script>` over wrapped components. `next/script` with `strategy="afterInteractive"` (the default) loads too late and misses Performance Timing data. Raw inline `<script>` in `<head>` is the simplest correct path.

## Cookies and consent

WireBoard ships three storage modes. Pick based on whether the site has a consent banner.

| Mode                  | When to use                                                                                 | What's stored on the visitor                                       |
| --------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Cookie mode** (default) | No consent banner needed (legitimate interest jurisdictions), or site already has implicit consent | `_wb_id` cookie (2y) + `_wb_ses` cookie (30min) + localStorage fallback |
| **Cookieless mode**       | Privacy-first site, no banner, or as the pre-consent default in a banner flow              | Nothing. In-memory only; each page load is a new visitor.          |
| **Banner flow** (recommended for GDPR/CCPA) | Site has a consent banner: load cookieless, upgrade on accept                  | Nothing pre-consent; cookies + localStorage after the user accepts |

### Cookieless mode (static config)

Use when the site has no consent banner and you want zero device storage. Swap the `newTracker` call:

```javascript
wireboard('newTracker', 'wb', 'pipeline-0.collector.wireboard.io', {
    appId: 'AbCd1234',
    forceSecureTracker: true,
    useCookies: false,
    useLocalStorage: false,
    contexts: { performanceTiming: true }
});
```

In cookieless mode each page load is treated as a new visitor (no returning-visitor detection). Pageviews, sessions within the page lifecycle, events, geographic data, device data, and referrer data all still work.

### Consent banner flow (the typical pattern)

For sites with a consent banner, do this:

**Step 1**: Initialize the tracker in **cookieless mode** (snippet above) so the page loads without setting any cookies before the user has consented.

**Step 2**: When the user accepts analytics cookies in your banner, call `upgradeStorage` to switch the live tracker to cookie mode mid-session:

```javascript
window.wireboard('upgradeStorage', { useCookies: true, useLocalStorage: true });
```

This sets `_wb_id` and `_wb_ses` from that point forward and enables returning-visitor detection. No page reload required. The visitor's current session continues seamlessly.

**Step 3 (revoke)**: There is no documented `downgradeStorage` API to mirror the upgrade direction, and the WireBoard cookies (`_wb_id`, `_wb_ses`) are set by the collector on its own domain, so the site's JavaScript cannot reliably delete them. The realistic revoke pattern is:

1. **Store the user's consent state in your CMP** (Cookiebot, OneTrust, Termly, or your own banner).
2. **Gate the `newTracker` initialization on that state**, so subsequent page loads initialize cookieless when consent is withdrawn:

   ```javascript
   const hasConsent = window.Cookiebot?.consent?.statistics === true; // adapt to your CMP

   wireboard('newTracker', 'wb', 'pipeline-0.collector.wireboard.io', {
       appId: 'AbCd1234',
       forceSecureTracker: true,
       useCookies: hasConsent,
       useLocalStorage: hasConsent,
       contexts: { performanceTiming: true }
   });
   ```

3. **Reload the page when the user withdraws consent mid-session.** The reload re-runs your `newTracker` config, which now sees `hasConsent === false` and initializes in cookieless mode. From that point on, no new cookies are set and no new tracking data is stored on the device.

   ```javascript
   function onConsentWithdrawn() {
       // Your CMP has already updated its consent state.
       // Reload so the tracker re-initializes in cookieless mode.
       window.location.reload();
   }
   ```

4. **Already-set cookies expire naturally**: `_wb_ses` after 30 minutes, `_wb_id` after 2 years. Some browsers and CMPs offer first-party cookie clearing for the collector domain; if yours does, use it as part of the withdrawal handler. Don't write your own cross-domain cookie deletion code, it will silently fail.

> Note: WireBoard exposes `window.wireboardClearPublisher()`, which clears the in-memory publisher ID used to attribute custom events. It does NOT delete cookies or change the tracker's storage mode. Do not use it as a consent-withdrawal handler; it solves a different problem (account-level identity, not device storage).

### Integration with common consent platforms

The pattern is always the same: **start cookieless, call `upgradeStorage` when consent is granted, call your revoke function when consent is withdrawn.**

**Cookiebot / OneTrust / Termly / Osano**: subscribe to their consent-change event and dispatch based on the analytics/statistics category:

```javascript
// Pseudocode, adapt to your provider's API
window.addEventListener('CookiebotOnAccept', () => {
    if (Cookiebot.consent.statistics) {
        window.wireboard?.('upgradeStorage', { useCookies: true, useLocalStorage: true });
    }
});
window.addEventListener('CookiebotOnDecline', () => {
    // Reload so newTracker re-initializes in cookieless mode (see "Step 3 (revoke)" above).
    window.location.reload();
});
```

**Custom banner**: call `upgradeStorage` directly from your Accept handler. The order in `<head>` matters: the WireBoard snippet must load first (in cookieless mode) so `window.wireboard` exists by the time your banner script runs.

### Default behavior

WireBoard defaults to cookie mode (full storage). Cookieless and the banner flow are explicit opt-ins.

> **Legal disclaimer**: WireBoard does not provide legal advice. Privacy regulations (GDPR, ePrivacy, CCPA, etc.) vary by jurisdiction. The "load cookieless, upgrade on consent" pattern above is the technical implementation for a typical GDPR-compliant flow; whether you need it, and what counts as valid consent, depends on your jurisdiction and the legal review of your consent banner. Consult a qualified legal professional.

## Verify

After install, ask the user to open their site in a fresh browser tab. The WireBoard dashboard's site detail page will show "Tracking installed" once it receives the first hit (usually within seconds).

For a DevTools-based check, see Step 5 of the main workflow in [`SKILL.md`](SKILL.md): filter the Network tab by collector hostname, look for HTTP 200 responses.
