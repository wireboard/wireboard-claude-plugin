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

Two options. Use manual import for now; the Community Gallery listing is still under review at the upstream repo.

### Option A: Manual import (works today)

1. Give the user this download link:

   ```text
   https://github.com/wireboard/wireboard-gtm-template/releases/latest/download/template.tpl
   ```

   This URL always resolves to the latest release's `template.tpl` asset. No need to look up the version.

2. In GTM: **Templates** → **New** in the **Tag Templates** section.
3. In the Template Editor, click the ⋮ menu (top right) → **Import**.
4. Select the downloaded `template.tpl`.
5. Save the template.

### Option B: Community Template Gallery (once approved)

The template is under review for the GTM Community Template Gallery. Once it's listed, this becomes the simpler path:

1. Open the GTM workspace.
2. **Templates** → in the **Tag Templates** section, click **Search Gallery**.
3. Search for **WireBoard** and click **Add to workspace**.
4. Review the permissions prompt and accept.

If the gallery search returns no result, the listing is not live yet. Fall back to Option A.

## Step 2: Create the Initialization tag

This tag loads WireBoard on every page and starts tracking. **Create exactly one** of these.

1. **Tags** → **New** → **Tag Configuration** → **WireBoard Analytics**.
2. Set **Tag Type** to **Initialization**.
3. Fill in the fields:

   | Field | Value |
   | --- | --- |
   | Site ID | The user's `appId` from their tracking snippet (8-char). Ask them to paste their snippet or open [the site manager](https://wireboard.io/dashboard?action=site-edit). |
   | Publisher ID | UUID from [Settings → Integrations](https://wireboard.io/dashboard/settings/integrations). Also embedded in the snippet. |
   | Collector URL | Leave the default (`pipeline-0.collector.wireboard.io`) unless the snippet shows a different host. |
   | Tracking mode | **Cookie** (default) or **Cookieless**. Match the user's consent posture. If they have a consent banner, see "Consent banner flow" below. |
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

If the site has a cookie consent banner, set the Initialization tag's **Tracking mode** to **Cookieless** so the page loads with no cookies before the user has consented.

The GTM template does not expose an `upgradeStorage` field on its Init tag, and firing a second Init tag on accept is not a pattern documented by the upstream template. For the upgrade-on-accept flow, call `window.wireboard('upgradeStorage', { useCookies: true, useLocalStorage: true })` directly from your banner's accept handler (this works because the Init tag has loaded `wireboard.js`, which exposes `window.wireboard` globally). The full handler patterns for Cookiebot, OneTrust, Termly, Osano, and custom banners are in [`tracking-script.md`](tracking-script.md) under "Integration with common consent platforms" — they are the same regardless of how `wireboard.js` was loaded.

For revoke, follow the reload-into-cookieless pattern from `tracking-script.md` Step 3. The CMP's consent state determines the Init tag's effective mode on the next page load.

## Step 4: Publish

1. **Submit** the workspace changes.
2. Use **Preview** to test before publishing: open the site in the Preview pane, check that the **WireBoard Analytics** Initialization tag fires on page load.

## Step 5: Verify

1. Open the site in a fresh browser tab.
2. DevTools → Network → filter by the **collector hostname** from the Initialization tag (default `pipeline-0.collector.wireboard.io`).
3. Reload. There should be at least one request to that hostname returning HTTP 200.
4. If Auto-capture is ON: trigger a tracked event (e.g. submit a form that already pushes to dataLayer) and confirm a second request appears.
5. The WireBoard dashboard's site detail page will show "Tracking installed" once the first hit lands.

## Out of scope for this path

- **SPA route tracking.** GTM's `Initialization - All Pages` trigger only fires on full page loads. If the site is a single-page app and the user needs per-navigation pageviews, use the code-edit install path (`tracking-script.md` + `spa-routing.md`) instead of, or alongside, this GTM path.
- **Inspecting `wireboard.js` or `events.min.js`.** Same rule as the rest of this skill: do not read tracker source. The fields above are the API contract.
