# Tracking script (`wireboard.js`)

The script that records pageviews, sessions and visitors. One per site.

## Snippet anatomy

The user pastes this from their WireBoard dashboard. Do NOT write one with placeholder IDs. If they ask where to find it:

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

| Line                            | Purpose                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------- |
| First block                     | Loads `wireboard.js` asynchronously from the CDN. Defines `window.wireboard` at once, so calls made before the script arrives are queued. |
| `newTracker`                    | Starts the tracker for this site: collector hostname, `appId`, HTTPS, and the performance timing context. |
| `enableActivityTracking(5, 10)` | Engagement heartbeat (5 s minimum, then every 10 s). Powers visit duration and bounce rate.                |
| `customContext`                 | Carries the Publisher UUID with the pageview. Every hit needs it.                                          |
| `trackPageView`                 | Sends the pageview for the page that just loaded.                                                         |

The `appId` (8-character site ID), the Publisher UUID and the collector hostname are specific to the user's site. Never invent any of them.

### Paste it exactly as given

Keep the snippet byte for byte, including `appId: '...'` with **single quotes** and one space after the colon. The dashboard's "connected" indicator works by fetching the site's homepage and looking for that exact text. Requoting it (`appId: "..."`), minifying it, or injecting it from a bundled JS file leaves the site shown as not connected even though tracking works.

When the framework needs the script body as a string (Next.js `__html`, Nuxt `innerHTML`), use a **template literal** (backticks) so the single quotes inside stay untouched.

### Storage options in the pasted snippet

Look at the `newTracker` options the user pasted:

- No `useCookies` / `useLocalStorage` options: **cookie mode** (the default, see "Cookies and consent" below).
- `useCookies: false` alone: no cookies, but the tracker still keeps a persistent visitor ID in local storage (`wb_duid`). Snippets from the dashboard's "Use cookies" switch looked like this before November 2026 (the switch now adds both options). It is NOT storage-free. If the user wants no device storage at all, add `useLocalStorage: false` as well, and tell them why.
- `useCookies: false` and `useLocalStorage: false`: cookieless mode.

## Where to install it

Place the snippet inside `<head>`, as early as possible. It loads asynchronously, so it does not block rendering.

| Framework | Target file | How |
| --------- | ----------- | --- |
| Plain HTML | every page's `<head>`, or the shared header partial | Paste the snippet. |
| Next.js (App Router) | `app/layout.tsx`, in the root layout's `<head>` | `<script dangerouslySetInnerHTML={{ __html: \`...\` }} />` inside `<head>`, the script body in a template literal. |
| Next.js (Pages Router) | `pages/_document.tsx`, inside `<Head>` | Create `_document.tsx` if missing. Import `Head` from `next/document` (not `next/head`). Same `dangerouslySetInnerHTML` script. |
| Nuxt 4 / Nuxt 3 | `nuxt.config.ts`, `app.head.script` | `[{ innerHTML: \`...\`, tagPosition: 'head' }]`. Use `innerHTML`, not `textContent` or `children`. |
| SvelteKit | `src/app.html` | Paste inside `<head>`, before `%sveltekit.head%`. |
| React Router v7/v8 framework mode (formerly Remix) | `app/root.tsx` | `<script dangerouslySetInnerHTML={{ __html: \`...\` }} />` inside `<head>` of the root `Layout`. Remix v2 projects (imports from `@remix-run/*`) use the same file. |
| Astro | the main layout (e.g. `src/layouts/Layout.astro`) | Paste inside `<head>` with `is:inline` so Astro does not bundle it: `<script is:inline type="text/javascript">...</script>`. |
| Vite (React, Vue, Svelte, Solid) | `index.html` | Paste inside `<head>`. |
| Angular | `src/index.html` | Paste inside `<head>`. |
| Laravel (Blade) | the layout's `<head>` (often `resources/views/layouts/app.blade.php`) | Paste. For consent and SPA navigation in one package, suggest [`wireboard/laravel-cmp`](https://github.com/wireboard/laravel-cmp) (see below). |
| Laravel + Inertia | the root view, usually `resources/views/app.blade.php` | Paste in `<head>`, then add SPA route tracking (Inertia navigates client-side). |
| Rails | `app/views/layouts/application.html.erb` | Paste in `<head>`. Rails 7 and later use Turbo Drive by default, which navigates client-side: add SPA route tracking (`turbo:load`). |
| Django | `base.html` | Paste in `<head>`. No `{% load static %}` needed, the URL is absolute. |
| WordPress | Recommend the [WireBoard Analytics plugin](https://wordpress.org/plugins/wireboard-analytics). | Manual fallback: the theme's `header.php` before `</head>`, or a header-code plugin. Never edit `wp-config.php` or core files. The WordPress plugin does not track logged-in WordPress users by default: test logged out or in a private window. |
| Wix / Squarespace | the platform's custom code panel, header section | Not a code change. Tell the user where to paste it. |

Prefer a raw inline `<script>` over wrapper components (`next/script`, Nuxt `useHead`): `next/script` with the default `afterInteractive` strategy loads late and misses the page's performance timing.

**Then check whether the site navigates client-side.** React, Vue, Svelte, Angular, Next.js, Nuxt, SvelteKit, React Router, Astro with `<ClientRouter />`, Inertia, Turbo Drive and Livewire `wire:navigate` all change pages without loading a new document. The snippet only counts the first page. See [`spa-routing.md`](spa-routing.md).

### Laravel: `wireboard/laravel-cmp`

For Laravel sites, WireBoard maintains [`wireboard/laravel-cmp`](https://github.com/wireboard/laravel-cmp) (Packagist): a consent banner that loads WireBoard cookieless first, upgrades on consent, erases WireBoard's cookies and local storage on withdrawal, and turns Inertia, Turbo and Livewire navigations into pageviews. Offer it when the Laravel site needs a consent banner or navigates client-side; installing the snippet by hand stays a valid choice.

## Cookies and consent

What the tracker stores depends on the storage options:

| Mode | When to use | What is stored on the visitor's device |
| ---- | ----------- | -------------------------------------- |
| **Cookie mode** (default) | No consent banner needed, or consent already given | Two first-party cookies on the site's own domain: `_wb_id.` + 4 characters (visitor ID, 2 years requested; browsers such as Chrome cap cookies at 400 days) and `_wb_ses.` + 4 characters (session, 30 minutes). Plus local storage: `wireboard_step`, `wireboard_last_request`, `wireboard_last_url` and the send queue `wireboardOutQueue_*`. |
| **Cookieless mode** | Privacy-first site without a banner, or the pre-consent state of a banner flow | No cookies and no tracker storage. Each page load counts as a new visitor (no returning-visitor detection). |
| **Banner flow** (typical for GDPR) | The site has a consent banner | Cookieless until the visitor accepts, then cookie mode from that moment (plus `wb_duid` in local storage). |

The 4 characters after `_wb_id.` and `_wb_ses.` are derived from the site's domain: they differ from site to site and never change for a given domain. The cookies are host-only (the page's exact host name), `Path=/`, not `HttpOnly`, written by the script in the page with `SameSite=Lax`, and `Secure` on HTTPS pages (tracker 1.2.2.3 and later). Safari limits cookies written by scripts to 7 days. The collector sets no cookies. Because they are first-party, the site's own scripts and its consent tool can read and delete them.

If `events.min.js` is loaded, it also saves the landing page's UTM parameters in local storage (keys `<host with dots as _>_utm_campaign`, `_utm_source`, and so on) while the tracker stores data (cookie or local storage mode, or after `upgradeStorage`). In cookieless mode it keeps them in memory for the page only (events.min.js 1.3.4 and later). Mention the keys when the user documents their storage for a privacy policy.

### Cookieless mode (static config)

Swap the `newTracker` call:

```javascript
wireboard('newTracker', 'wb', 'pipeline-0.collector.wireboard.io', {
    appId: 'AbCd1234',
    forceSecureTracker: true,
    useCookies: false,
    useLocalStorage: false,
    contexts: { performanceTiming: true }
});
```

Pageviews, sessions within the page, events, geography, devices and referrers still work.

### Consent banner flow

**Step 1**: start the tracker in cookieless mode (snippet above), so nothing is stored before consent.

**Step 2**: when the visitor accepts analytics, switch the running tracker to cookie mode. No reload needed. Only do this while the tracker is cookieless: on later page loads the tracker must already start in cookie mode (Step 3, point 1), because a tracker that started cookieless ignores the existing cookie and `upgradeStorage` would then replace the visitor's ID on every page:

```javascript
window.wireboard('upgradeStorage', { useCookies: true, useLocalStorage: true });
```

**Step 3 (withdrawal)**: there is no method to switch back. On withdrawal, erase what consent allowed, then reload so `newTracker` starts cookieless again:

1. Gate the `newTracker` options on the consent state, so every page load starts in the right mode:

   ```javascript
   const hasConsent = window.Cookiebot?.consent?.statistics === true; // adapt to your consent tool

   wireboard('newTracker', 'wb', 'pipeline-0.collector.wireboard.io', {
       appId: 'AbCd1234',
       forceSecureTracker: true,
       useCookies: hasConsent,
       useLocalStorage: hasConsent,
       contexts: { performanceTiming: true }
   });
   ```

2. On withdrawal, delete the cookies and local storage entries, then reload:

   ```javascript
   function eraseWireboardData() {
       document.cookie.split(';').forEach((c) => {
           const name = c.split('=')[0].trim();
           if (/^_wb_/.test(name)) document.cookie = name + '=; Max-Age=0; path=/';
       });
       const utmPrefix = location.hostname.replace(/\./g, '_') + '_utm_';
       try {
           for (let i = localStorage.length - 1; i >= 0; i--) {
               const key = localStorage.key(i);
               if (/^(wb_duid|wireboard_(step|last_request|last_url))$|^wireboardOutQueue/.test(key) || key.indexOf(utmPrefix) === 0) {
                   localStorage.removeItem(key);
               }
           }
       } catch (e) { /* storage unavailable: nothing was stored */ }
   }

   function onConsentWithdrawn() {
       eraseWireboardData();
       window.location.reload();
   }
   ```

   If the consent tool has its own cookie clearing (for example an `autoClear` list), add the pattern `^_wb_` to it. Consent tools usually clear cookies only, so keep the local storage part.

Erase these entries only on consent withdrawal. Never clear them as a "fix" for tracking problems: it breaks visit and journey data.

> `window.wireboardClearPublisher()` only clears the publisher ID used for custom events. It does not delete storage or change the storage mode: do not use it for withdrawal.

### Consent platforms

The pattern is always: **start cookieless, `upgradeStorage` on accept, erase and reload on withdrawal.**

```javascript
// Cookiebot example, adapt the event names to your provider
window.addEventListener('CookiebotOnAccept', () => {
    if (Cookiebot.consent.statistics) {
        window.wireboard?.('upgradeStorage', { useCookies: true, useLocalStorage: true });
    }
});
window.addEventListener('CookiebotOnDecline', () => {
    onConsentWithdrawn(); // erase, then reload into cookieless mode
});
```

OneTrust, Termly, Osano and custom banners: subscribe to their consent-change callback the same way. Load the WireBoard snippet first in `<head>` so `window.wireboard` exists when the banner script runs.

> **Legal note**: WireBoard does not give legal advice. Whether a banner is needed and what counts as valid consent depends on the jurisdiction (GDPR, ePrivacy, CCPA, and others). The pattern above is the technical implementation of a typical consent flow; the user should have it checked by a qualified professional.

## Verify

Follow Step 5 of the main workflow in [`SKILL.md`](SKILL.md). Test in a normal, visible browser: headless browsers are filtered out as bots.
