# Install via Google Tag Manager

For sites that already use GTM. WireBoard ships an official template that loads the tracker, configures cookies, and forwards `dataLayer` events without any code changes to the site.

Source: [github.com/wireboard/wireboard-gtm-template](https://github.com/wireboard/wireboard-gtm-template). Apache 2.0. Two tag types: **Initialization** and **Custom Event**.

## When to use this path

This is an **alternative** install path, not a requirement. Even when GTM is already on the site, the direct code snippet install (see `tracking-script.md`) remains a perfectly valid choice and is sometimes preferable (full control over load order, simpler SPA route tracking, no GTM publish step).

Pick the GTM path when:

- The user explicitly prefers managing analytics from GTM rather than the codebase.
- The user does not have access to deploy the codebase but does have GTM access (marketing, ops).
- The user already manages other tags through GTM and wants WireBoard alongside them.

Do not push GTM users onto the template by default. Ask which path they want.

If the site does not use GTM, do not push them to install GTM just for WireBoard. The direct snippet install is simpler and faster.

## Step 1: Install the template into GTM

Two options. Manual import always works; the dashboard's own install guide uses it.

### Option A: Manual import

1. Give the user this download link:

   ```text
   https://github.com/wireboard/wireboard-gtm-template/releases/latest/download/template.tpl
   ```

   This URL always resolves to the latest release's `template.tpl` asset. No need to look up the version.

2. In GTM: **Templates** → **New** in the **Tag Templates** section.
3. In the Template Editor, click the ⋮ menu (top right) → **Import**.
4. Select the downloaded `template.tpl`.
5. Save the template.

### Option B: Community Template Gallery

If the template is listed in the GTM Community Template Gallery, this is quicker:

1. Open the GTM workspace.
2. **Templates** → in the **Tag Templates** section, click **Search Gallery**.
3. Search for **WireBoard** and click **Add to workspace**.
4. Review the permissions prompt and accept.

If the gallery search returns no result, use Option A.

## Step 2: Create the Initialization tag

This tag loads WireBoard on every page and starts tracking. **Create exactly one** of these.

1. **Tags** → **New** → **Tag Configuration** → **WireBoard Analytics**.
2. Set **Tag Type** to **Initialization**.
3. Fill in the fields:

   | Field | Value |
   | --- | --- |
   | Site ID | The user's `appId` from their tracking snippet (8-char). Ask them to paste their snippet or open [the site manager](https://wireboard.io/dashboard?action=site-edit). |
   | Publisher ID | The UUID in the user's tracking snippet (`publisher:'...'`). Settings, Integrations shows the selected team owner's UUID, which is wrong if another team is selected. |
   | Collector URL | Leave the default (`pipeline-0.collector.wireboard.io`) unless the snippet shows a different host. |
   | Tracking mode | **Cookie** (default), **Cookieless**, or **Consent Mode** (template v1.1.0 and later). With a consent banner, see "Consent banner flow" below. |
   | Auto-capture events from window.dataLayer (recommended) | Leave **ON** if the site already pushes GA4-style events to `window.dataLayer`. Forwards them to WireBoard automatically. |
   | Track engagement (activity heartbeat) | Leave **ON** (heartbeat: time on page, scroll, etc.). |

4. **Triggering** → select the built-in **Initialization - All Pages** trigger.
5. Save.

## Step 3: Custom Event tags (optional)

Skip this if the site already pushes events to `window.dataLayer` and Auto-capture is ON. Those events forward automatically. Use Custom Event tags only when the user wants events fired by a GTM trigger that is not already pushed to `dataLayer`.

1. **Tags** → **New** → **WireBoard Analytics** → **Tag Type: Custom Event**.
2. Fields:

   | Field | Required | Notes |
   | --- | --- | --- |
   | Category | Yes | e.g. `ecommerce`, `engagement`. |
   | Action | Yes | e.g. `purchase`, `sign_up`. |
   | Label | No | Optional identifier. |
   | Value | No | Numeric (e.g. price). |
   | Custom properties | No | Key/value table, up to 30 entries. |

   Do NOT set Publisher ID here. It is inherited from the Initialization tag.

3. **Triggering** → attach to whatever GTM trigger represents the event (a click trigger, form submit trigger, etc.).
4. Save.

For the event field semantics (what category vs action vs label means, value normalization, props rules, the 30/300/2000 limits, reserved keys, PII rules), see [`custom-events.md`](custom-events.md). The same rules apply whether the event comes from `wireboardEvent()`, the dataLayer auto-capture, or a GTM Custom Event tag.

## Consent banner flow with GTM

**Do not combine the Cookieless mode with `upgradeStorage` in the banner's accept handler.** Consent tools announce an existing consent again on every page load, and a tracker that starts cookieless ignores the visitor's cookie: each `upgradeStorage` then writes a new visitor ID, so a returning visitor counts as new on every page.

Pick one setup with the user:

1. **Consent Mode (recommended with a banner)**: Tracking mode **Consent Mode (cookieless until analytics_storage is granted)**, available from template v1.1.0. When the tag fires it reads Google Consent Mode's `analytics_storage`: visitors who already consented are tracked with cookies and keep their visitor ID, the others are tracked cookieless, and the tracker switches to cookies once if the visitor accepts during the page. Everyone is counted. Requirements: the consent tool feeds Google Consent Mode (Cookiebot, OneTrust, Usercentrics and most others can), and the tag fires on **Initialization - All Pages** without "Require additional consent" (it must fire for visitors who decline too). If the container still has template v1.0.0, update it first: download `template.tpl` from the link in Step 1, open the template in GTM, **⋮**, **Import**, save.
2. **Track only after consent**: Tracking mode **Cookie**, and in the tag's **Advanced Settings, Consent Settings**, choose **Require additional consent for tag to fire** with `analytics_storage`. Nothing is tracked before consent; visitors who never consent are not counted.
3. **Count everyone, never store anything**: Tracking mode **Cookieless**, no `upgradeStorage`. No returning-visitor detection.

For withdrawal, use the erase-then-reload handler from `tracking-script.md` (Step 3). After the reload, setup 1 starts cookieless again, setup 2 no longer fires, and setup 3 stored nothing.

## Step 4: Preview, then publish

1. Click **Preview**, open the site in the Tag Assistant window, and check that the **WireBoard Analytics** Initialization tag fires on page load.
2. When it does, click **Submit** and publish the workspace version. Nothing reaches visitors until it is published.

## Step 5: Verify

Follow Step 5 of [`SKILL.md`](SKILL.md) (visible browser, Network tab filtered by the collector hostname, then confirm the visit in WireBoard). If Auto-capture is ON, also trigger an event the site already pushes to `dataLayer` and confirm a second request appears.

The site manager's "connected" icon stays off for GTM installs: it looks for the snippet in the homepage's HTML, and GTM adds it later in the browser. That is expected; confirm with real traffic instead.

## Single-page apps with GTM

The Initialization tag runs on full page loads only, so client-side navigations are not counted out of the box. Two ways to fix it:

**A. In the codebase**: add the `<meta name="wireboard-publisher">` tag and the router hook from [`spa-routing.md`](spa-routing.md). The router hook reads the publisher from the meta tag; it does not get it from GTM.

**B. In GTM only**: a Custom HTML tag fired by a **History Change** trigger. Use the same Publisher UUID as the Initialization tag:

```html
<script>
(function () {
  var strip = function (u) { return String(u).split('#')[0]; };
  var nav = window.performance && performance.getEntriesByType && performance.getEntriesByType('navigation')[0];
  var last = window.__wbLastUrl || strip(nav ? nav.name : document.referrer);
  var url = strip(location.href);
  if (url === last || typeof window.wireboard !== 'function') return; // same page, or tracker not loaded
  window.__wbLastUrl = url;
  setTimeout(function () { // give the app time to set the new page title
    window.wireboard('setCustomUrl', url);
    window.wireboard('setReferrerUrl', last);
    window.wireboard('trackPageView', null, [
      { schema: 'wb:io.wireboard/publisher', data: { publisher: 'YOUR-PUBLISHER-UUID' } }
    ]);
  }, 250);
})();
</script>
```

The History Change trigger also fires for `#anchor` jumps and URL rewrites that do not change the page; the tag ignores those. Check it in Preview: one WireBoard request per real navigation.
