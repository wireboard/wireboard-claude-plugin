# SPA route tracking on client-side navigation

The tracking snippet calls `trackPageView` exactly once, when the script first loads on the initial document. SPAs do client-side navigation (`/home` to `/pricing` without a full reload) and the snippet does not fire a new pageview for those. You must wire it up explicitly.

**Without this wiring, the dashboard shows one pageview per session instead of one per route.**

## Prerequisites (REQUIRED, even if not using custom events)

The SPA route wiring reads the Publisher UUID from a `<meta>` tag. **Add this to `<head>` even when the user does not want custom events**:

```html
<meta name="wireboard-publisher" content="YOUR-PUBLISHER-UUID">
```

The UUID is the same value embedded in the tracking snippet's `customContext`. Extract it from what the user pasted, or ask.

Without this meta tag, every SPA navigation produces a failed HTTP 400 request to the collector.

## The reusable pattern

Every framework example below reads the publisher from the meta tag and calls `trackPageView` with the required context array:

```javascript
const publisher = document
    .querySelector('meta[name="wireboard-publisher"]')
    ?.getAttribute('content');
if (!publisher) return; // guard: missing meta tag would produce HTTP 400
const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
window.wireboard?.('trackPageView', null, ctx);
```

The optional `?.` chain handles ad blockers and slow script loads gracefully.

## Per-framework wiring

### Next.js App Router

Create a client component and mount it once in the root layout. Depend on `pathname` only, NOT `useSearchParams()` (that hook requires a `<Suspense>` boundary in production builds; the build fails with `Missing Suspense boundary with useSearchParams` otherwise, and you usually do not want a new pageview on every query-param tweak anyway).

```tsx
// app/wireboard-route-tracker.tsx
'use client';
import { useEffect, useRef } from 'react';
import { usePathname } from 'next/navigation';

export function WireboardRouteTracker() {
    const pathname = usePathname();
    const first = useRef(true);
    useEffect(() => {
        if (first.current) { first.current = false; return; } // snippet already counted page 1
        const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
        if (!publisher) return;
        const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
        window.wireboard?.('trackPageView', null, ctx);
    }, [pathname]);
    return null;
}
```

Then in `app/layout.tsx`:

```tsx
<body>
    <WireboardRouteTracker />
    {children}
</body>
```

If query-param-aware pageviews are needed (rare for analytics), add `useSearchParams()` AND wrap the tracker in `<Suspense fallback={null}>` where it is mounted.

### Next.js Pages Router

In `pages/_app.tsx`. `routeChangeComplete` does NOT fire on initial mount, so no first-render guard:

```tsx
import type { AppProps } from 'next/app';
import { useRouter } from 'next/router';
import { useEffect } from 'react';

export default function MyApp({ Component, pageProps }: AppProps) {
    const router = useRouter();
    useEffect(() => {
        const handler = () => {
            const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
            if (!publisher) return;
            const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
            window.wireboard?.('trackPageView', null, ctx);
        };
        router.events.on('routeChangeComplete', handler);
        return () => router.events.off('routeChangeComplete', handler);
    }, [router]);
    return <Component {...pageProps} />;
}
```

### React Router v6

Drop a tracker component inside the `<BrowserRouter>` subtree. Depend on `pathname` only (not `search`):

```jsx
import { useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';

function WireboardRouteTracker() {
    const location = useLocation();
    const first = useRef(true);
    useEffect(() => {
        if (first.current) { first.current = false; return; }
        const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
        if (!publisher) return;
        const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
        window.wireboard?.('trackPageView', null, ctx);
    }, [location.pathname]);
    return null;
}
```

### Vue Router

In `main.ts` after creating the router. `afterEach` fires on initial navigation too, so guard:

```ts
let first = true;
router.afterEach(() => {
    if (first) { first = false; return; }
    const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
    if (!publisher) return;
    const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
    window.wireboard?.('trackPageView', null, ctx);
});
```

### Nuxt 3

Create `plugins/wireboard-route.client.ts` (the `.client.ts` suffix makes it browser-only):

```ts
export default defineNuxtPlugin(() => {
    const router = useRouter();
    let first = true;
    router.afterEach(() => {
        if (first) { first = false; return; }
        const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
        if (!publisher) return;
        const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
        (window as any).wireboard?.('trackPageView', null, ctx);
    });
});
```

### SvelteKit

In `src/routes/+layout.svelte`. `afterNavigate` fires on initial component mount AND on every navigation, so guard:

```svelte
<script>
import { afterNavigate } from '$app/navigation';
let first = true;
afterNavigate(() => {
    if (first) { first = false; return; }
    const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
    if (!publisher) return;
    const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
    window.wireboard?.('trackPageView', null, ctx);
});
</script>
```

### Remix

In `app/root.tsx`. Uses React Router internally so the pattern matches:

```tsx
import { useLocation } from '@remix-run/react';
import { useEffect, useRef } from 'react';

function WireboardRouteTracker() {
    const location = useLocation();
    const first = useRef(true);
    useEffect(() => {
        if (first.current) { first.current = false; return; }
        const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
        if (!publisher) return;
        const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
        window.wireboard?.('trackPageView', null, ctx);
    }, [location.pathname]);
    return null;
}
```

Mount `<WireboardRouteTracker />` inside the `<Layout>` component in `app/root.tsx`.

### Astro with client-side routing (`<ClientRouter />` / `<ViewTransitions />`)

Astro is mostly multi-page (full reloads), so the snippet's initial `trackPageView` covers most usage. For pages using client-side routing, wire to `astro:after-swap`.

In Astro 5 and later the routing component is `<ClientRouter />` (imported from `astro:transitions`). In Astro 3 and 4 it was `<ViewTransitions />`; that name still works as a deprecated alias on Astro 5. The event name (`astro:after-swap`) is the same on both.

`astro:after-swap` only fires on transitions (not on initial full-page load), so no first-render guard is needed. The defensive `if (!window.__wbAstroInit)` guard makes the listener registration idempotent across any unusual execution path:

```html
<script is:inline>
if (!window.__wbAstroInit) {
    window.__wbAstroInit = true;
    document.addEventListener('astro:after-swap', () => {
        const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
        if (!publisher) return;
        const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
        window.wireboard?.('trackPageView', null, ctx);
    });
}
</script>
```

### Vanilla SPA (History API)

Listen on `popstate` and patch `pushState` / `replaceState`. Run once on initial load (the snippet's own `trackPageView` covers the first hit, so no extra call):

```javascript
function fireWireboardPV() {
    const publisher = document.querySelector('meta[name="wireboard-publisher"]')?.getAttribute('content');
    if (!publisher) return;
    const ctx = [{ schema: 'wb:io.wireboard/publisher', data: { publisher } }];
    window.wireboard?.('trackPageView', null, ctx);
}
const origPush = history.pushState;
history.pushState = function (...args) { origPush.apply(this, args); fireWireboardPV(); };
const origReplace = history.replaceState;
history.replaceState = function (...args) { origReplace.apply(this, args); fireWireboardPV(); };
window.addEventListener('popstate', fireWireboardPV);
```

## Why the customContext is REQUIRED on every call

WireBoard rejects any tracking request whose customContext is missing a `wb:io.wireboard/publisher` entry. The response is HTTP 400 `publisher not found`. When the `(appId, publisher)` pair does not match a real site on the user's account, the response is HTTP 403 `invalid publisher or appid`.

A `trackPageView` call without the context is silently dropped from the user's perspective: no console error from the tracker, but every SPA navigation produces a failed request. Always include the context array on every call.

The `if (!publisher) return;` guard in each pattern above is defense-in-depth: it skips the call entirely when the meta tag is missing, so no failed requests get sent.
