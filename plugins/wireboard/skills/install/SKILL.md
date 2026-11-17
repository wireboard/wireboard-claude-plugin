---
name: install
description: Install WireBoard analytics in a codebase, or debug an existing WireBoard integration that isn't working. Covers the tracking script for pageviews, client-side (SPA) navigation tracking, custom events (declarative or programmatic), cookie consent, the official Google Tag Manager template, the dataLayer bridge, and a step-by-step troubleshooting flow.
when_to_use: |
  Use when the user mentions: install wireboard, add wireboard, wireboard.io, wireboardEvent, data-wireboard-event, wireboard tracking, wireboard SPA, wireboard cookie consent, set up custom events, track button click with wireboard, wireboard via GTM, wireboard tag manager template. ALSO use for debugging: events aren't firing, no events, events not working, wireboard not tracking, pageviews missing, wireboard dashboard empty, tracking not working, wireboardEvent does nothing, site not connected. Do NOT use for general analytics unrelated to WireBoard. Questions about the user's traffic numbers are answered with the WireBoard MCP tools, not this skill.
argument-hint: "[install | events | debug]"
---

# WireBoard integration

## Working rules

1. **Never invent values.** The `appId`, the collector hostname and the Publisher UUID come from the user's WireBoard dashboard. Ask the user to paste their snippet; never write one with placeholder IDs.
2. **Paste the snippet unchanged.** Same quotes, same spacing (see [`tracking-script.md`](tracking-script.md), "Paste it exactly as given").
3. **In DevTools, filter by the collector hostname** (the value in the user's `newTracker` call, e.g. `pipeline-0.collector.wireboard.io`). The request paths do not matter for debugging.
4. **Call the script "the WireBoard tracker" or `wireboard.js`.**
5. **Use this skill as the API reference.** It documents every option and message a developer needs; reading the minified `wireboard.js` or `events.min.js` does not help.
6. **Test in a visible browser.** Headless browsers (Playwright or Puppeteer in headless mode) are filtered as bots: their hits get HTTP 200 and are never counted. If you test with a browser tool, launch it with `headless: false`, or ask the user to click through the site.
7. **Always verify after implementing** (Step 5). Don't call it done before a real visit shows up in WireBoard.

## What WireBoard ships

Two scripts, both from `https://static.wireboard.io/`:

- **`wireboard.js`**: the tracking script. Pageviews, sessions, visitors. One per site.
- **`events.min.js`**: optional. Custom events (clicks, form submits, purchases).

Two identifiers the user must provide (never invent them):

| Identifier | Format | Where to get it |
| ---------- | ------ | --------------- |
| **Site ID (`appId`)** | 8 characters (e.g. `AbCd1234`) | In the tracking snippet, on the site's Install page |
| **Publisher UUID** | UUID | In the same snippet (`publisher:'...'` in `customContext`). Settings, Integrations also shows one, but it is the selected team owner's: with another team selected it is wrong, and every hit is refused with HTTP 403. Prefer the snippet. |

## Workflow

Follow this in order. Details for each step are in the linked file.

### Step 1: Scope and framework

Ask the user:
- Tracking only, custom events only, or both?
- Do they have a WireBoard site yet? If yes, the snippet is in the site manager: [wireboard.io/dashboard?action=site-edit](https://wireboard.io/dashboard?action=site-edit), select the site, click **Install**. If they have no account: [wireboard.io/register](https://wireboard.io/register) (free plan available), then create the site.

Detect the framework from `package.json`, `composer.json`, `Gemfile`, `nuxt.config.*`, `astro.config.*`, `next.config.*`, `angular.json` or the project root. Confirm with the user before editing.

### Step 2: Install the tracking script

**If the site already uses Google Tag Manager**, offer the choice: the official WireBoard GTM template (no code change, managed in GTM) or the code snippet. Neither is required just because GTM is present. Detect GTM by searching for `googletagmanager.com/gtm.js` or a `GTM-` container ID, or ask. For the GTM path see [`gtm.md`](gtm.md) and skip the rest of this step.

Ask the user to paste their snippet and add it, unchanged, to the right place for their framework.

**Ask whether the site has a cookie consent banner.** If it does, the tracker should start cookieless and switch to cookies on consent, with erase-and-reload on withdrawal. Also check the storage options in the pasted snippet: `useCookies: false` alone still stores a visitor ID in local storage.

Read [`tracking-script.md`](tracking-script.md) for the snippet anatomy, the install location per framework (Next.js, Nuxt, SvelteKit, React Router, Astro, Vite, Angular, Laravel and Inertia, Rails, Django, plain HTML, WordPress, Wix, Squarespace), and the storage modes and consent flow.

### Step 3: Client-side navigation (required when the site navigates without full page loads)

The snippet counts only the first page. Single-page apps and client-side navigation need explicit wiring: React and Next.js, Vue and Nuxt, SvelteKit, Angular, React Router, TanStack Router, Astro with `<ClientRouter />`, Laravel Inertia, Rails with Turbo Drive (the Rails 7+ default), and Livewire `wire:navigate`.

You must do BOTH:
1. Add `<meta name="wireboard-publisher" content="UUID">` to `<head>`, even if the user wants no custom events.
2. Add the shared helper and call it from the router hook, as shown in [`spa-routing.md`](spa-routing.md).

Without the meta tag, navigations are silently not counted (no request, no error).

**GTM path and a single-page app**: the GTM Initialization tag runs on full page loads only. Either add the meta tag and router hook to the codebase, or add the Custom HTML tag on a History Change trigger described in [`gtm.md`](gtm.md).

### Step 4: Custom events (only if wanted)

1. Add `<script src="https://static.wireboard.io/events.min.js"></script>` to `<head>`, after the main snippet.
2. Add `<meta name="wireboard-publisher" content="UUID">` if not already there.
3. Instrument the events, in one of two styles:
   - **Declarative**: `data-wireboard-event-*` attributes, for static HTML and templates.
   - **Programmatic**: `window.wireboardEvent({...})`, for React, Vue, Svelte, Angular and any component that mounts and unmounts.

Read [`custom-events.md`](custom-events.md) before writing event code: fields, value normalization, props rules, the 30/300/2000 limits, reserved keys, PII rules, casing, attribute names, the API signature, framework examples and the GTM dataLayer mirror. The API takes `{category, action, label, value, props}`, NOT `{name, props}`.

### Step 5: Verify

1. Open the site in a normal, visible browser window (a private window without extensions is best). Not a headless browser.
2. DevTools, Network tab, filter by the **collector hostname**. Reload: at least one request to it should return HTTP 200. For a single-page app, navigate: one more request per page.
3. For custom events, trigger one: an extra request appears.
4. **A 200 alone does not prove the visit was counted** (bots and IPs on the IP Ignore List also get 200). Confirm in WireBoard:
   - a new site's onboarding has a **Verify Installation** step that shows "Traffic Detected!" when the first visit arrives;
   - or the dashboard's real-time view shows the visit within seconds;
   - or, if the WireBoard MCP tools are connected (see below), call `get_live_state` for the site (paid plans) or `get_aggregate` with range `today`.
5. After testing, the user can add their own IP to the site's IP Ignore List so their visits stop counting. Not before: it would hide the test visits.

The site manager's "connected" icon is a separate check of the homepage HTML: it stays off for GTM installs and for snippets added through JavaScript, even when tracking works.

If something fails, go to [`troubleshooting.md`](troubleshooting.md) rather than improvising: the WireBoard-specific failure modes (publisher context, meta tag, bot filter, quota) are listed there.

## WireBoard analytics tools (MCP)

This plugin also connects Claude to the user's WireBoard data through the WireBoard MCP server: sites, period totals, time series, daily history, breakdowns (country, device, referrer, page...), top URLs, custom event queries and the live snapshot. All tools are read-only.

They need an API token. If a tool answers "No WireBoard API token is set", tell the user:

1. Create a token at [wireboard.io/dashboard/settings/api](https://wireboard.io/dashboard/settings/api) with the `analytics:read` ability (and `live:read` for the live snapshot).
2. In Claude Code, run `/plugin`, open the **Installed** tab, select **WireBoard**, choose **Configure options**, and paste the token. Then run `/reload-plugins` or restart Claude Code.

The live snapshot needs a paid plan; on the free plan, history queries cover the last 30 days. The tools say so when a limit applies.

## Reference files

Load these when the step needs them. Quote only small parts in replies.

- [`tracking-script.md`](tracking-script.md): snippet anatomy, install location per framework, storage modes, consent flow
- [`gtm.md`](gtm.md): install through the official Google Tag Manager template, consent with GTM, single-page apps with GTM
- [`spa-routing.md`](spa-routing.md): the shared navigation helper and its wiring per framework
- [`custom-events.md`](custom-events.md): event fields, declarative attributes, programmatic API, value normalization, props rules and limits, framework examples, dataLayer mirror
- [`troubleshooting.md`](troubleshooting.md): diagnostic flow, collector responses, console messages, symptom table

## Not covered here

- Building your own integration in code: the [`@wireboard/api`](https://www.npmjs.com/package/@wireboard/api) (JavaScript/TypeScript) and [`wireboard-api`](https://pypi.org/project/wireboard-api/) (Python) SDKs and the [API docs](https://wireboard.io/docs/api-overview).
- General analytics not specific to WireBoard.
