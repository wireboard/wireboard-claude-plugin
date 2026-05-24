# Troubleshooting

When the WireBoard integration isn't working ("events aren't firing", "no pageviews", "dashboard empty"), walk the user through this diagnostic flow IN ORDER. Do NOT propose reading `wireboard.js` or `events.min.js` source; the API contract is documented in this skill's reference files and the symptom table below covers every known failure mode.

## Quick diagnostic flow ("nothing is working")

Ask the user to open DevTools and check in this order. Don't skip steps.

### 1. Is the snippet actually on the page?

In the browser, view source (Ctrl+U / Cmd+U) on the affected page. Search for `wireboard tag` or `wireboard.io`. If it's not there:
- The snippet was never added, OR
- The build pipeline stripped it (check the prod-vs-dev section below)

### 2. Is the tracking script loading?

DevTools, **Network tab**, filter by `wireboard.js`. Reload the page. You should see one request to `https://static.wireboard.io/wireboard.js` returning HTTP 200. If not:
- 404 / blocked: an extension (ad blocker, privacy tool) is blocking `static.wireboard.io`. Test in a private window with no extensions.
- Not loaded at all: the snippet isn't running because it's outside `<head>`, or it's in a code path that didn't execute (server-only rendering, etc.).

### 3. Are tracking requests going out?

DevTools, **Network tab**, filter by the **collector hostname** from the `newTracker` call in the snippet (e.g. `pipeline-0.collector.wireboard.io`, or `127.0.0.1:8080` for local dev). Reload the page. You should see at least one request to that hostname.

- **No requests at all**: the snippet ran but `trackPageView` didn't fire. Check the console (next step).
- **Request shows HTTP 400 `publisher not found`**: missing publisher customContext. The snippet's `customContext` block is wrong or the SPA route hook is calling `trackPageView` without context. See "HTTP 400" in the symptom table.
- **Request shows HTTP 403 `invalid publisher or appid`**: the `(appId, publisher)` pair doesn't match a real site on the user's account. Check both values against the WireBoard dashboard.
- **Request shows HTTP 200**: tracking is working at the network layer. If the dashboard still appears empty, see "Events accepted but dashboard empty" in the symptom table.

### 4. Any console errors?

DevTools, **Console tab**, filter by `wireboard`. Look for any of:
- `Invalid publisher ID provided`: publisher UUID is missing or empty. Add or fix the `<meta name="wireboard-publisher">` tag, or pass `publisher:` per call.
- `Valid publisher is required`: same issue, different code path (custom events).
- `Both category and action are required`: a `wireboardEvent()` call is missing one of the two required fields.
- `prop "..." collides with UTM namespace`: a prop key used one of the reserved UTM names. Rename it.
- `Session event limit reached`: 1000 events per session has been hit. Reload to reset.

### 5. Is `wireboardEvent` actually defined?

In the **Console tab**, type:

```javascript
window.wireboardEvent
```

If it returns `undefined`, `events.min.js` never loaded. Check:
- The `<script src="https://static.wireboard.io/events.min.js"></script>` line is in `<head>`
- It's AFTER the main tracking snippet, not before
- It returns HTTP 200 (Network tab, filter `events.min.js`)

If it returns a function, the API is ready.

### 6. Can you fire an event manually?

In the Console tab:

```javascript
window.wireboardEvent({ category: 'test', action: 'click' })
```

- Returns `true` → call was accepted. Check the collector network filter for the resulting request.
- Returns `false` → look in console for the rejection reason and find it in the symptom table.

### 7. Is the meta tag present?

In the Console tab:

```javascript
document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content')
```

If it returns `null`, the meta tag isn't in `<head>`. Every event call will be rejected with `Valid publisher is required`. Add the tag.

If all seven pass and the dashboard still appears empty, the calls ARE landing client-side: see "Events accepted but dashboard empty" below.

## Symptom table

| Symptom                                                                          | Likely cause                                                                                                                          | Fix                                                                                                                                                                                                                                                                                       |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dashboard "Tracking installed" status never appears                              | Snippet not in `<head>`, page cached, or ad blocker                                                                                   | View source to confirm snippet is in served HTML. Clear CDN/site cache. Test in a private window without extensions.                                                                                                                                                                      |
| SPA shows one pageview per session even though users navigate a lot              | Client-side route changes don't auto-fire `trackPageView`                                                                              | Wire up the router hook from [`spa-routing.md`](spa-routing.md) (`usePathname`, `routeChangeComplete`, `afterEach`, `afterNavigate`, etc.).                                                                                                                                               |
| SPA route changes produce HTTP 400 `publisher not found` in Network tab          | `trackPageView` called without the publisher customContext                                                                            | Pass the customContext array `[{schema:'wb:io.wireboard/publisher', data:{publisher}}]` on every call. WireBoard rejects requests without it. See [`spa-routing.md`](spa-routing.md) for the pattern.                                                                                     |
| SPA route changes produce HTTP 403 `invalid publisher or appid`                  | `appId` and Publisher UUID mismatch (e.g. snippet's appId is for site A, meta tag's publisher is from a different account)            | Verify the `<meta name="wireboard-publisher">` UUID matches the publisher embedded in the tracking snippet's `customContext`.                                                                                                                                                              |
| Next.js production build fails with "Missing Suspense boundary with useSearchParams" | Used `useSearchParams()` in the route tracker without a Suspense boundary                                                              | Drop `useSearchParams` and depend only on `usePathname`. If query-param tracking is required, wrap the tracker in `<Suspense fallback={null}>`. See [`spa-routing.md`](spa-routing.md).                                                                                                   |
| SvelteKit fires double pageview on initial load                                  | Forgot the first-render guard on `afterNavigate` (it fires on initial mount, contrary to some assumptions)                            | Add `let first = true; if (first) { first = false; return; }` guard. See [`spa-routing.md`](spa-routing.md).                                                                                                                                                                               |
| Astro accumulates duplicate pageview events after a few transitions              | Listener registered multiple times                                                                                                    | Wrap the `addEventListener` in `if (!window.__wbAstroInit) { window.__wbAstroInit = true; ... }` so it registers exactly once. See [`spa-routing.md`](spa-routing.md).                                                                                                                      |
| `wireboardEvent is not a function` in console                                    | `events.min.js` not loaded, or loaded before `wireboard.js`                                                                            | Check the script src loads (Network tab). Ensure `events.min.js` is AFTER the main snippet in `<head>`.                                                                                                                                                                                    |
| Console error: `Invalid publisher ID provided` or `Valid publisher is required`  | Publisher UUID missing or empty                                                                                                       | Add `<meta name="wireboard-publisher" content="UUID">` in `<head>`, or pass `publisher:` per `wireboardEvent()` call.                                                                                                                                                                      |
| Console warning: `prop "..." collides with UTM namespace`                        | Used `utm_source`, `utm_campaign`, etc. as a prop key                                                                                  | Rename the prop. UTMs are captured automatically when `utm: true` (the default). Override with `utm: false` per event if needed.                                                                                                                                                          |
| Custom event shows category and action but no label                              | Label is optional; this is expected                                                                                                   | Add `label` if you want a third axis to filter by.                                                                                                                                                                                                                                        |
| Same property splits into two keys (`cartSize` and `cartsize`)                   | Mixed declarative and JS forms for the same key (HTML lowercases, JS preserves case)                                                  | Pick one casing per key and use it everywhere. Recommended: lowercase with hyphens.                                                                                                                                                                                                         |
| View events fire dozens of times                                                 | Declarative `data-wireboard-event="view"` on a remounting SPA component                                                                | Switch to programmatic `window.wireboardEvent` inside `useEffect` with `useInView({ triggerOnce: true })`. See [`custom-events.md`](custom-events.md).                                                                                                                                    |
| Value shows as `0.00` in the dashboard                                           | Non-numeric string passed as `value`                                                                                                  | Use a number, or move the dimension to `props`. See "Value normalization" in [`custom-events.md`](custom-events.md).                                                                                                                                                                       |
| Events fire but only sometimes                                                   | Ad blocker on test machine                                                                                                            | Test in a clean profile without extensions.                                                                                                                                                                                                                                                |
| `wireboardEvent(...)` called, returns `true`, but nothing in the dashboard       | Either propagation latency (events take seconds to surface), OR the request is being dropped client-side by an extension              | Wait ~30 seconds and refresh. Open Network tab filtered by collector hostname; should see one request per call returning HTTP 200. If no request at all → ad blocker. If request returns 4xx → check the HTTP 400 / 403 rows above.                                                       |
| `wireboardEvent(...)` returns `false` but no obvious console error               | Session event cap (1000) reached, or no publisher resolvable                                                                          | Check console for `Valid publisher is required`, `Invalid publisher ID provided`, or `Session event limit reached`. Reload the page (resets session counter). Verify the meta tag is present.                                                                                              |
| Tracking works in dev but fails in prod                                          | Build pipeline stripped the snippet from `<head>`, OR CSP blocks the script src, OR prod environment has stricter ad blockers          | Check the prod page's served HTML (view source). Check the Content-Security-Policy response header for restrictions on `static.wireboard.io` or the collector hostname.                                                                                                                    |
| Events accepted but dashboard empty                                              | Propagation latency, dashboard filter view, or wrong dashboard scope                                                                  | Wait ~60 seconds. Check the dashboard's date filter includes "today". Check you're looking at the right site in the dashboard sidebar.                                                                                                                                                     |
