# Pageviews on client-side navigation

The snippet sends one pageview, for the document that loaded. Single-page apps and client-side navigation (`/home` to `/pricing` without loading a new document) send nothing more on their own.

**Without this wiring the dashboard shows one pageview per visit instead of one per page.**

This applies to: React (React Router, TanStack Router), Next.js, Vue Router, Nuxt, SvelteKit, Angular, Astro with `<ClientRouter />`, Laravel Inertia, Rails Turbo Drive, Livewire `wire:navigate`, and any app that calls `history.pushState`.

## Prerequisite: the publisher meta tag

Every pageview needs the Publisher UUID. The navigation code reads it from a meta tag, so add this to `<head>` even if the user does not want custom events:

```html
<meta name="wireboard-publisher" content="YOUR-PUBLISHER-UUID">
```

Use the UUID from the snippet the user pasted (`publisher:'...'` in `customContext`). Without the tag the helper below sends nothing: navigations are silently not counted, with no error anywhere.

## The shared helper

Put this once in the app (for example `src/lib/wireboard.ts`, or an inline `<script>` for non-bundled sites) and call `trackWireboardRoute()` from the router hook. It:

- sends one pageview per real address change, so router hooks that also fire on first render, repeated events for one navigation, `replaceState` calls that do not navigate, and `#anchor` jumps are all ignored;
- waits briefly for the new page's `<title>` (frameworks set it while the page mounts), so a view never carries the previous page's title;
- reports the new address and the previous page as referrer.

```ts
// wireboard.ts: one WireBoard pageview per client-side navigation.
const strip = (href: string) => href.split('#')[0];
let lastUrl = strip(window.location.href);

export function trackWireboardRoute(): void {
    const url = strip(window.location.href);
    if (url === lastUrl) return; // same page: first render, duplicate event, hash jump
    const referrer = lastUrl;
    lastUrl = url;

    const titleBefore = document.title;
    let sent = false;
    const send = () => {
        if (sent) return;
        sent = true;
        observer.disconnect();
        clearTimeout(timer);
        const publisher = document
            .querySelector('meta[name="wireboard-publisher"]')
            ?.getAttribute('content');
        if (!publisher || typeof window.wireboard !== 'function') return;
        window.wireboard('setCustomUrl', url);
        window.wireboard('setReferrerUrl', referrer);
        window.wireboard('trackPageView', document.title, [
            { schema: 'wb:io.wireboard/publisher', data: { publisher } },
        ]);
    };
    // Wait for the title to change, at most 250 ms (pages that keep their title).
    const observer = new MutationObserver(() => {
        if (document.title !== titleBefore) setTimeout(send, 0);
    });
    observer.observe(document.head, { childList: true, subtree: true, characterData: true });
    const timer = setTimeout(send, 250);
}
```

For plain JavaScript, drop the type annotation and `export`, and expose it as `window.trackWireboardRoute = trackWireboardRoute`. For TypeScript, declare `window.wireboard` once (see "TypeScript" in [`custom-events.md`](custom-events.md)).

Hash routers (`/#/dashboard` style): use the full `window.location.href` instead of `strip(...)`, because there the fragment is the page address.

**Performance timing on SPAs**: the snippet's `contexts: { performanceTiming: true }` describes the first document load and is attached to every later pageview of the visit, which skews page-load metrics for client-side views. For an SPA, suggest removing that option from `newTracker` (or setting it to `false`) and explain why; it is the user's choice.

## Per-framework wiring

Each pattern below just calls the helper. No first-render guard is needed: the helper ignores a call for the page already counted.

### Next.js App Router

A client component mounted once in the root layout. Depend on `pathname` only: `useSearchParams()` needs a `<Suspense>` boundary or the production build fails with `Missing Suspense boundary with useSearchParams`.

```tsx
// app/wireboard-route-tracker.tsx
'use client';
import { useEffect } from 'react';
import { usePathname } from 'next/navigation';
import { trackWireboardRoute } from '@/lib/wireboard';

export function WireboardRouteTracker() {
    const pathname = usePathname();
    useEffect(() => { trackWireboardRoute(); }, [pathname]);
    return null;
}
```

In `app/layout.tsx`: `<body><WireboardRouteTracker />{children}</body>`.

### Next.js Pages Router

In `pages/_app.tsx`:

```tsx
import type { AppProps } from 'next/app';
import { useRouter } from 'next/router';
import { useEffect } from 'react';
import { trackWireboardRoute } from '../lib/wireboard';

export default function MyApp({ Component, pageProps }: AppProps) {
    const router = useRouter();
    useEffect(() => {
        router.events.on('routeChangeComplete', trackWireboardRoute);
        return () => router.events.off('routeChangeComplete', trackWireboardRoute);
    }, [router]);
    return <Component {...pageProps} />;
}
```

### React Router (v7, v8) and React Router framework mode

React Router 8 removed the `react-router-dom` package: import from `react-router` (v7 accepts both). Inside the router tree, for example in the root layout or `app/root.tsx`:

```tsx
import { useLocation } from 'react-router';
import { useEffect } from 'react';
import { trackWireboardRoute } from './lib/wireboard';

export function WireboardRouteTracker() {
    const { pathname } = useLocation();
    useEffect(() => { trackWireboardRoute(); }, [pathname]);
    return null;
}
```

Older projects: React Router v6 imports `useLocation` from `react-router-dom`, Remix v2 from `@remix-run/react`. Match what the project already imports.

### TanStack Router / TanStack Start

After creating the router:

```ts
router.subscribe('onResolved', () => trackWireboardRoute());
```

### Vue Router

In `main.ts`, after creating the router:

```ts
router.afterEach(() => trackWireboardRoute());
```

### Nuxt

Nuxt 4 keeps app code in `app/`: create `app/plugins/wireboard-route.client.ts` (Nuxt 3: `plugins/wireboard-route.client.ts`). The `.client.ts` suffix makes it browser-only. Put the helper in `app/utils/wireboard.ts` (Nuxt 3: `utils/`) so it is auto-imported, or import it.

```ts
export default defineNuxtPlugin(() => {
    useRouter().afterEach(() => trackWireboardRoute());
});
```

### SvelteKit

In `src/routes/+layout.svelte`:

```svelte
<script>
    import { afterNavigate } from '$app/navigation';
    import { trackWireboardRoute } from '$lib/wireboard';

    afterNavigate(() => trackWireboardRoute());
</script>
```

### Angular

In the root component:

```ts
import { Component, inject } from '@angular/core';
import { NavigationEnd, Router } from '@angular/router';
import { trackWireboardRoute } from './wireboard';

@Component({ /* ... */ })
export class AppComponent {
    constructor() {
        inject(Router).events.subscribe((e) => {
            if (e instanceof NavigationEnd) trackWireboardRoute();
        });
    }
}
```

### Astro with `<ClientRouter />`

Astro is multi-page unless the layout uses `<ClientRouter />` (from `astro:transitions`; Astro 3 and 4 called it `<ViewTransitions />`, a name removed since). With it, listen to `astro:after-swap`, in an inline script in the layout, together with the helper as plain JavaScript:

```html
<script is:inline>
if (!window.__wbAstroInit) {
    window.__wbAstroInit = true;
    document.addEventListener('astro:after-swap', () => window.trackWireboardRoute?.());
}
</script>
```

### Laravel Inertia, Rails Turbo, Livewire

These announce navigations with document events. Load the helper as plain JavaScript (in the main JS entry or an inline script) and add the matching listener:

```javascript
document.addEventListener('inertia:navigate', () => trackWireboardRoute());   // Inertia
document.addEventListener('turbo:load', () => trackWireboardRoute());         // Rails Turbo Drive
document.addEventListener('livewire:navigated', () => trackWireboardRoute()); // Livewire wire:navigate
```

They also fire on the first page load: the helper ignores that call. For Laravel, [`wireboard/laravel-cmp`](https://github.com/wireboard/laravel-cmp) does all of this (plus consent) without custom code.

### Any other SPA (History API)

```javascript
const origPush = history.pushState;
history.pushState = function (...args) { origPush.apply(this, args); trackWireboardRoute(); };
const origReplace = history.replaceState;
history.replaceState = function (...args) { origReplace.apply(this, args); trackWireboardRoute(); };
window.addEventListener('popstate', () => trackWireboardRoute());
```

Routers often call `replaceState` without changing the page: the helper ignores those.

## Why every pageview needs the publisher context

The collector rejects a pageview whose context has no `wb:io.wireboard/publisher` entry: HTTP 400 `publisher not found`. If the `appId` and publisher do not belong together (for example the meta tag holds another team's UUID), the answer is HTTP 403 `invalid publisher or appid`. Always pass the context array, as the helper does.
