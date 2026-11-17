# Troubleshooting

When the integration seems broken ("events aren't firing", "no pageviews", "dashboard empty"), walk the user through this flow in order. The API contract is in this skill's reference files, so there is no need to read the minified `wireboard.js` or `events.min.js`.

## First: how are you testing?

Ask this before anything else. Two things make working tracking look broken:

- **Headless or automated browsers are filtered as bots.** Visits from headless Chrome (Playwright or Puppeteer in headless mode, Lighthouse, crawlers) get an HTTP 200 like any other hit and are never counted. That is on purpose. Test in a normal, visible browser; with Playwright or Puppeteer, launch with `headless: false`. If you (Claude) want to test with a browser tool, use a visible one or ask the user to click through the site.
- **IP Ignore List.** If the tester's IP is on the site's IP Ignore List (site settings), their hits also get a 200 and are dropped. Changes to the list take up to 5 minutes. Advise adding one's own IP only after testing is done.

Also: the WordPress plugin does not track logged-in WordPress users by default. Test logged out or in a private window.

## Diagnostic flow

### 1. Is the snippet on the page?

View source (Ctrl+U / Cmd+U) and search for `wireboard tag` or `static.wireboard.io`. If missing, the snippet was never added or the build removed it (see "Works in dev, not in prod" below).

### 2. Is the tracking script loading?

DevTools, **Network** tab, filter `wireboard.js`, reload. Expect one request to `https://static.wireboard.io/wireboard.js` with HTTP 200.
- Blocked or failed: an extension (ad blocker, privacy tool) or a Content-Security-Policy is blocking `static.wireboard.io`. Test in a private window without extensions; check the CSP header.
- Not requested at all: the snippet did not run (outside `<head>`, server-only code path, or a script-blocking consent tool).

### 3. Are hits reaching the collector?

Network tab, filter by the **collector hostname** from the `newTracker` call (e.g. `pipeline-0.collector.wireboard.io`), reload.

| Response | Meaning | Fix |
| -------- | ------- | --- |
| No request | The snippet ran but sent nothing | Check the console (step 4). For client-side navigations, check the router wiring and the publisher meta tag ([`spa-routing.md`](spa-routing.md)). |
| HTTP 200 | Received. Counted, unless the visitor is a bot or headless browser, or the IP is on the IP Ignore List (see above) | If the dashboard stays empty, see "Hits return 200 but nothing appears" below. |
| HTTP 400 `publisher not found` | The pageview had no publisher context | Pass the `customContext` array on every `trackPageView` call ([`spa-routing.md`](spa-routing.md)). |
| HTTP 400 `appid is required` / `context is required` | The snippet was edited and lost its `appId` or its context | Paste the original snippet again, unchanged. |
| HTTP 403 `invalid publisher or appid` | The `appId` and Publisher UUID do not belong together | Use the UUID from the site's own snippet. The one in Settings, Integrations belongs to the selected team's owner and is wrong when another team is selected. |
| HTTP 403 `quota exceeded` | The plan's monthly pageview quota is used up | Check usage in the dashboard's settings (Usage); upgrade the plan or wait for the next period. Nothing is wrong with the code. |
| HTTP 403 `forbidden` | The request was refused before processing | Check the snippet is unmodified and the site exists in the dashboard. If it persists, contact WireBoard support. |
| HTTP 415, 500, 503 | A problem on WireBoard's side, usually temporary | Try again a little later. If it lasts, contact WireBoard support. |

### 4. Console messages

DevTools, **Console** tab. Do not filter by "wireboard": most messages do not contain that word. Look for:

| Message | Meaning |
| ------- | ------- |
| `Invalid publisher ID provided` | The `<meta name="wireboard-publisher">` tag, or a `wireboardSetPublisher()` call, holds an empty or malformed UUID. |
| `Valid publisher is required` | A custom event had no publisher: add the `<meta name="wireboard-publisher">` tag or pass `publisher:`. |
| `Both category and action are required` | A `wireboardEvent()` call is missing one of the two. |
| `wireboardEvent: prop "..." collides with UTM namespace` | A prop used a reserved `utm_*` key. Rename it. |
| other `wireboardEvent: ...` warnings | Props over the limits (30 keys, 300-character keys, 2000-character values) or non-scalar values. See [`custom-events.md`](custom-events.md). |
| `Invalid or unreasonable number detected` | A `value` could not be read as a number and was sent as `0.00`. |
| `Session event limit reached, dropping event` | 1000 custom events in this session. Reload to reset. |
| `Wireboard not available, requeueing event` | `wireboard.js` is not loaded (blocked, or the snippet is missing). Events wait for it. |
| `Circuit breaker active, dropping event` / `Circuit breaker active. Next attempt in ...` | Sending failed repeatedly (network or blocker); events are paused for a while. |

### 5. Is `wireboardEvent` defined?

In the console: `window.wireboardEvent`. If `undefined`, `events.min.js` did not load: check the `<script src="https://static.wireboard.io/events.min.js"></script>` tag is present and returns HTTP 200 (Network tab). Its order relative to the snippet does not matter for this.

### 6. Fire an event by hand

```javascript
window.wireboardEvent({ category: 'test', action: 'click' })
```

- `true`: queued. A request to the collector hostname should follow (step 3).
- `false`: rejected. The console says why (step 4).

### 7. Is the publisher meta tag present?

```javascript
document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content')
```

`null` means the tag is missing: custom events are refused with `Valid publisher is required`, and client-side navigations are not counted at all (no request, no error).

## Hits return 200 but nothing appears

In order:

1. **Bot filter or IP Ignore List** (see the top of this page). The most common cause when an AI agent or a script did the testing.
2. **Wrong place or period**: the right site selected in the dashboard, the date filter includes today.
3. **Delay**: live views update within seconds; reports can take a minute.
4. **Quota**: a 403 `quota exceeded` on later hits means the plan's quota ran out mid-period.

## Symptom table

| Symptom | Likely cause | Fix |
| ------- | ------------ | --- |
| The site manager shows the site as not connected, but traffic arrives | The indicator checks the homepage HTML for the exact text `appId: '<site id>'`. GTM installs, snippets injected by a JS bundle, and requoted snippets (`appId: "..."`) never match | Expected for GTM. Otherwise paste the snippet unchanged into the HTML. Rely on real traffic (Verify step in `SKILL.md`). |
| One pageview per visit although visitors navigate | Client-side navigation not wired | [`spa-routing.md`](spa-routing.md). Also on Rails (Turbo Drive), Inertia and Livewire `wire:navigate`. |
| Client-side navigations not counted, no errors, no requests | Publisher meta tag missing | Add `<meta name="wireboard-publisher" content="UUID">` to `<head>`. |
| Navigations produce HTTP 400 `publisher not found` | Custom router code calls `trackPageView` without the context | Use the helper in [`spa-routing.md`](spa-routing.md). |
| HTTP 403 `invalid publisher or appid` | UUID from another team, or `appId` of another site | Take both from the site's own snippet. |
| HTTP 403 `quota exceeded` | Plan quota used up | Usage in the dashboard settings; upgrade or wait. |
| Two pageviews for one navigation | Several hooks for one navigation, or custom code without de-duplication | Use only the helper from [`spa-routing.md`](spa-routing.md), called from one hook. |
| Pageviews show the previous page's title | `trackPageView` sent before the app set the new title | Use the helper (it waits for the title). |
| Next.js build fails with `Missing Suspense boundary with useSearchParams` | `useSearchParams()` in the route tracker | Depend on `usePathname` only, or wrap the tracker in `<Suspense fallback={null}>`. |
| Astro counts each navigation several times | Listener registered more than once | Keep the `window.__wbAstroInit` guard ([`spa-routing.md`](spa-routing.md)). |
| `wireboardEvent is not a function` | `events.min.js` not loaded or blocked | Check its script tag and its network request. |
| Custom event has category and action but no label | Label is optional | Add `label` if a third level is wanted. |
| One property split into two keys (`cartSize` and `cartsize`) | Declarative and JS forms mixed (HTML lowercases attribute names) | One casing per key everywhere; lowercase with hyphens is safest. |
| View events fire many times | Declarative `data-wireboard-event="view"` on a component that remounts | Programmatic `wireboardEvent` with `useInView({ triggerOnce: true })` ([`custom-events.md`](custom-events.md)). |
| Value shows `0.00` | No value, a non-numeric string, or a currency code (`CHF 10`) | Pass a number. Formats like `1.234,56` are also misread (`1.23`). See "Value normalization" in [`custom-events.md`](custom-events.md). |
| Works in dev, not in prod | Build or CMS removed the snippet, a Content-Security-Policy blocks `static.wireboard.io` or the collector, or a consent tool blocks scripts | View the production page source; check the CSP header (`script-src` for `static.wireboard.io`, `connect-src` and `img-src` for the collector hostname). |
| Test visits never show, real ones do | Headless test browser or IP Ignore List | See the top of this page. Never "fix" it by clearing `wireboard_*` local storage keys: that breaks visit and journey data. |
