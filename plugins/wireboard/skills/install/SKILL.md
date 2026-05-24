---
name: install
description: Install or debug WireBoard analytics in a codebase. Covers the tracking script for pageviews, SPA client-side route tracking, custom events (declarative or programmatic), and the GTM dataLayer bridge. Includes a troubleshooting flow for when things are not working.
when_to_use: |
  Use when the user mentions: install wireboard, add wireboard, wireboard.io, wireboardEvent, data-wireboard-event, wireboard tracking, wireboard SPA, set up custom events, track button click with wireboard. ALSO use for debugging: events aren't firing, no events, events not working, wireboard not tracking, pageviews missing, wireboard dashboard empty, tracking not working, wireboardEvent does nothing. Do NOT use for general analytics unrelated to WireBoard, or for the public REST/Live API (point users at the @wireboard/api or wireboard-api SDKs instead).
argument-hint: "[install | events | debug]"
---

# WireBoard Integration

## Hard rules (apply throughout this skill)

1. **Never fabricate values.** The `appId`, the per-site collector URL, and the Publisher UUID all come from the user's WireBoard dashboard. Ask the user to paste their snippet; never template one with placeholder IDs.
2. **Never invent collector request paths.** When asking the user to verify in DevTools Network tab, filter by **collector hostname** (the value from `newTracker`, e.g. `pipeline-0.collector.wireboard.io`, or `127.0.0.1:8080` for local dev). Never mention specific paths like `/i` or `/tp2`. The exact request path is an internal detail users do not need to see.
3. **Never name any upstream tracker library.** Refer to the script as "the WireBoard tracker" or `wireboard.js`. WireBoard is its own product.
4. **Never inspect tracker source** ("let me look at events.min.js to see what it expects"). This skill IS the API contract. The source is minified and reading it wastes the user's time. Use the reference files below.
5. **Never leak internal architecture** (queue sizes, retry counts, observer types, dispatch internals). Document only what a developer can act on.
6. **Always verify after implementing.** Don't declare done until the user has loaded their site, driven the tracked interaction, and confirmed they see a request to the collector hostname returning HTTP 200 (or you have walked them through this check).

## What WireBoard ships

Two scripts, both loaded from `https://static.wireboard.io/`:

- **`wireboard.js`**: the tracking script. Captures pageviews, sessions, visitors. One per site.
- **`events.min.js`**: optional. Adds custom event tracking (button clicks, form submits, purchases) account-wide.

Two identifiers the user must provide (never invent these):

| Identifier             | Format                                      | Where to get it                                                                   |
| ---------------------- | ------------------------------------------- | --------------------------------------------------------------------------------- |
| **Site UID (`appId`)** | 8-char ID (e.g. `AbCd1234`)                 | Embedded in the user's tracking snippet on their site's Install page              |
| **Publisher UUID**     | UUID v4                                     | Settings, Integrations: [wireboard.io/dashboard/settings/integrations](https://wireboard.io/dashboard/settings/integrations). Also embedded in the snippet. |

## Workflow

Follow this in order. Detailed reference for each step is in the linked file.

### Step 1: Confirm scope and detect the framework

Ask the user:
- Do they want tracking only, custom events only, or both?
- Have they already created a site in their WireBoard dashboard? If yes, point them to the site manager (opens the site modal directly): [wireboard.io/dashboard?action=site-edit](https://wireboard.io/dashboard?action=site-edit), select site, click Install to copy the snippet. (Alternative: [wireboard.io/dashboard](https://wireboard.io/dashboard), click Manage your Sites in the sidebar.) If they don't have a site yet, they need to create one first via the same dashboard.

Detect their framework by reading `package.json`, `composer.json`, `Gemfile`, `nuxt.config.*`, `astro.config.*`, `next.config.*`, or by listing the project root. Confirm with the user before editing.

### Step 2: Install the tracking script

Ask the user to paste their tracking snippet. **Paste it verbatim** into the right entry point for their framework.

**Also ask: does the site have a cookie consent banner?** If yes, the correct pattern is to load the tracker cookieless and upgrade on consent (not the default cookie-mode snippet). See the "Consent banner flow" section in [`tracking-script.md`](tracking-script.md). If no banner, paste the snippet as-is (default cookie mode) or switch to static cookieless if the site is privacy-first.

Read [`tracking-script.md`](tracking-script.md) for: the snippet anatomy (what each line does), the per-framework install table (Next.js App and Pages Router, Nuxt 3, SvelteKit, Remix, Astro, Vite, Laravel, Rails, Django, plain HTML, WordPress, Wix and Squarespace), and the three cookie modes (default cookie / static cookieless / banner flow with `upgradeStorage` + revoke).

### Step 3: SPA route tracking (REQUIRED for SPAs)

If the project is a SPA (React, Next.js, Vue, Nuxt, SvelteKit, Remix, or Astro with client-side routing via `<ClientRouter />` on Astro 5+ or `<ViewTransitions />` on Astro 3-4), the snippet only fires ONE pageview on initial load. Client-side navigations need explicit wiring.

For SPAs, you MUST do BOTH:
1. Add the `<meta name="wireboard-publisher" content="UUID">` tag to `<head>` (the SPA route wiring reads the UUID from there). Required even if the user does not want custom events.
2. Wire up the framework's router hook to call `trackPageView` on each navigation. Patterns per framework are in [`spa-routing.md`](spa-routing.md).

Skipping either step rejects every client-side pageview with HTTP 400 `publisher not found`.

### Step 4: Custom events (only if the user wants them)

If the user wants custom event tracking:

1. Add `<script src="https://static.wireboard.io/events.min.js"></script>` to `<head>`, AFTER the main tracking snippet.
2. Add `<meta name="wireboard-publisher" content="UUID">` if not already present.
3. Instrument the events. Two styles:
   - **Declarative**: `data-wireboard-event-*` HTML attributes. Use for static HTML and templates.
   - **Programmatic**: `window.wireboardEvent({...})` calls. **Use for React, Vue, Svelte, and any component that mounts and unmounts dynamically.**

Full anatomy (fields, value normalization, props rules, the 30/300/2000 limits, reserved keys, PII rules, casing consistency, declarative attribute names, programmatic API signature, framework examples, GTM dataLayer mirror) is in [`custom-events.md`](custom-events.md). Read it before writing any event code. The API uses `{category, action, label, value, props}`, NOT `{name, props}`.

### Step 5: Verify

Walk the user through this check:

1. Open the site in a fresh browser tab (preferably a private window with no extensions).
2. Open DevTools, Network tab, filter by their **collector hostname** (the value from the `newTracker` call in their snippet, e.g. `pipeline-0.collector.wireboard.io`).
3. Reload the page. They should see at least one request to that hostname returning HTTP 200.
4. If they're testing custom events, trigger one (click the tracked button, submit the tracked form). They should see an additional request.
5. The WireBoard dashboard's site detail page will show "Tracking installed" once it receives the first hit.

If anything doesn't work, go to [`troubleshooting.md`](troubleshooting.md). DO NOT improvise debugging steps from general analytics knowledge; the WireBoard-specific failure modes (HTTP 400 publisher gate, the SPA route hook, the meta tag requirement) are documented there.

## Reference files

Load these on demand for the work in each step. Do not copy their content into your reply unless quoting a small block.

- [`tracking-script.md`](tracking-script.md): per-framework install patterns + snippet anatomy + cookieless mode
- [`spa-routing.md`](spa-routing.md): client-side route tracking for React, Next, Vue, Nuxt, SvelteKit, Remix, Astro, vanilla History API
- [`custom-events.md`](custom-events.md): event anatomy, declarative attributes, programmatic API, value normalization, props rules and limits, casing, framework examples, dataLayer mirror
- [`troubleshooting.md`](troubleshooting.md): diagnostic flow when integration isn't working + symptom table

## What this skill does NOT cover

- The WireBoard public REST and Live (SSE) APIs. Point users at the [`@wireboard/api`](https://www.npmjs.com/package/@wireboard/api) (JS/TS) or [`wireboard-api`](https://pypi.org/project/wireboard-api/) (Python) SDKs and the [API docs](https://wireboard.io/docs/api-overview).
- The WireBoard MCP server. See [`@wireboard/mcp`](https://www.npmjs.com/package/@wireboard/mcp).
- General analytics not specific to WireBoard.
