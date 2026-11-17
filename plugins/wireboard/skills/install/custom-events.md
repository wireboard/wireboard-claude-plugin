# Custom events (`events.min.js`)

Tracks button clicks, form submissions, purchases, video plays, and any other user interaction. Optional. Requires the main tracking script (`wireboard.js`) to be installed first.

## Setup (one-time)

1. Add the events script AFTER the main snippet, in the same `<head>`:

   ```html
   <script src="https://static.wireboard.io/events.min.js"></script>
   ```

2. Add the publisher meta tag in `<head>`:

   ```html
   <meta name="wireboard-publisher" content="YOUR-PUBLISHER-UUID">
   ```

   Take the UUID from the snippet the user pasted (`publisher:'...'` in `customContext`). Settings, Integrations shows the UUID of the currently selected team's owner: with another team selected it is the wrong one, and every event is refused.

Without the meta tag, every tracked element must carry the Publisher UUID via `data-wireboard-event-<type>-publisher="UUID"`, `data-wireboard-event-publisher="UUID"`, or the legacy `data-wireboard-publisher="UUID"`. The meta tag is preferred.

## Event anatomy

Every event has these fields:

| Field      | Required | Type             | Description                                                                                          |
| ---------- | -------- | ---------------- | ---------------------------------------------------------------------------------------------------- |
| `category` | yes      | string           | Broad grouping. Examples: `ecommerce`, `engagement`, `signup`.                                       |
| `action`   | yes      | string           | What happened. Examples: `click`, `add-to-cart`, `purchase`, `form-submit`.                          |
| `label`    | no       | string           | Specific identifier. Examples: `header-cta`, `blue-t-shirt`, `pro-plan`.                             |
| `value`    | no       | string \| number | Numeric quantity. Normalized to a 2-decimal float string. See "Value normalization" below.           |
| `utm`      | no       | boolean          | Capture UTM params from current URL. Defaults to `true`.                                             |
| `props`    | no       | object           | Up to 30 key-value pairs. See "Custom properties" below.                                             |
| `publisher`| see note | string           | Publisher UUID. Required if no `<meta>` tag and no `data-wireboard-publisher` attribute on element.  |

**Required: `category` + `action`. Everything else is optional.**

## Value normalization

The `value` field is **always normalized to a 2-decimal string** before being sent. Pass a plain number whenever you can:

- `49.99` or `"49.99"` → `"49.99"`
- `"120"` → `"120.00"` (always 2 decimals)
- `"49,99"` → `"49.99"` (a comma followed by exactly 2 digits is read as a decimal comma)
- `"$1,234.56"` → `"1234.56"` (`$`, `€`, `£`, `¥` and thousands commas are removed)
- `"1.234,56"` → `"1.23"` (wrong: dot thousands separators are not understood)
- `"CHF 10"`, `"10 EUR"` and other currency codes → `"0.00"`
- no value, non-numeric, `NaN`, or larger than 1e9 → `"0.00"` (the non-numeric cases log `Invalid or unreasonable number detected`)

So convert prices to a number in code (`49.99`, not a formatted string). For non-currency quantities (seconds watched, items in cart, score), pass a number too. For non-numeric dimensions (color, size, plan tier), use `props`.

## Custom properties (`props`)

Props attach arbitrary dimensions to an event. The user can later filter and break events down by any prop in their WireBoard dashboard widgets.

**Hard limits** (enforced client-side with console warning AND at ingest):

| Rule                          | Limit                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------- |
| Max props per event           | 30 (excess silently dropped)                                                     |
| Max chars per key             | 300 (over-limit keys dropped)                                                    |
| Max chars per value           | 2000 (over-limit values truncated)                                               |
| Reserved keys (rejected)      | `utm_campaign`, `utm_source`, `utm_medium`, `utm_content`, `utm_term`            |
| Value types accepted          | strings, numbers, booleans (numbers and booleans coerced to string)              |
| Empty / whitespace-only       | Dropped. Keys and values are trimmed before storage.                             |
| Max events per session        | 1000. After that, calls return `false` and log a warning.                        |

**Never send PII in props.** No emails, names, addresses, phone numbers, precise locations, IPs, or IDs that map to a person. On the direct paths (declarative attributes and programmatic API), WireBoard does NOT auto-strip PII; that's the developer's responsibility. (The dataLayer mirror documented later has its own deny list, but that does NOT apply to direct calls.) Keep props to non-identifying dimensions (plan tier, variant, payment method, locale, cart size).

## Casing consistency between declarative and JS

This is the #1 prop bug. The two forms handle key casing differently:

- **Declarative HTML**: browsers lowercase attribute names. `data-wireboard-event-prop-cartSize` is stored as `cartsize`.
- **JavaScript API**: casing preserved. `props: { cartSize: 3 }` is stored as `cartSize`.

`cartSize` and `cartsize` are two different keys as far as the dashboard is concerned. Mixing both forms for the same property splits your data. **Pick one casing per key and use it everywhere.** Recommended: lowercase with hyphens (HTML forces it anyway).

## Style 1: Declarative (HTML data-attributes)

For static HTML, server-rendered templates, and any element that mounts once and stays mounted. **Do NOT use for SPA components that mount and unmount dynamically** (see "SPA pitfall" below).

### Event types

```html
data-wireboard-event="click"            <!-- fires on click -->
data-wireboard-event="view"             <!-- fires when 50% of element is in viewport, once per element instance -->
data-wireboard-event="load"             <!-- fires when element is added to the DOM -->
data-wireboard-event="click|view"       <!-- multiple, pipe-separated -->
```

`view` fires once per element instance. If the element is removed and a new one is added, the new one fires again. `load` and `view` both work on elements added after initial page render (via React, htmx swap, etc.); newly inserted elements are picked up automatically.

### Per-type vs general attributes

For each field (`category`, `action`, `label`, `value`, `utm`, `publisher`):

- **Type-specific**: `data-wireboard-event-click-category="..."` (applies only to click events on this element)
- **General fallback**: `data-wireboard-event-category="..."` (default for any event type when no type-specific value is set)

Specific wins over general when both are set.

**Props are different**: NO type infix. The same props apply to every event type on the element:

```
data-wireboard-event-prop-<key>="value"
```

### Examples

Single click tracking:

```html
<button
    data-wireboard-event="click"
    data-wireboard-event-click-category="ecommerce"
    data-wireboard-event-click-action="add-to-cart"
    data-wireboard-event-click-label="blue-t-shirt"
    data-wireboard-event-click-value="29.99"
>
    Add to Cart
</button>
```

Multiple types with shared category, different actions:

```html
<div
    data-wireboard-event="view|click"
    data-wireboard-event-category="content"
    data-wireboard-event-view-action="content-view"
    data-wireboard-event-click-action="content-click"
    data-wireboard-event-label="blog-post-42"
>
    Featured article
</div>
```

With custom properties:

```html
<button
    data-wireboard-event="click"
    data-wireboard-event-click-category="signup"
    data-wireboard-event-click-action="complete"
    data-wireboard-event-prop-plan="pro"
    data-wireboard-event-prop-variant="long-form"
    data-wireboard-event-prop-locale="fr"
>
    Sign up for Pro
</button>
```

Disable UTM capture per element:

```html
<button
    data-wireboard-event="click"
    data-wireboard-event-click-category="internal"
    data-wireboard-event-click-action="back-to-top"
    data-wireboard-event-utm="false"
>Back to top</button>
```

## Style 2: Programmatic (JavaScript API)

For SPAs (React, Vue, Svelte, Solid, Qwik), events triggered by non-DOM interactions (API responses, timers, scroll depth, websocket messages), and components that mount/unmount dynamically.

### The API

```javascript
window.wireboardEvent({
    category: 'user',           // required
    action: 'login',            // required
    label: 'success',           // optional
    value: '49.99',             // optional, normalized to 2 decimals
    utm: true,                  // optional, default true
    publisher: 'UUID',          // required if no <meta> tag
    props: {                    // optional, 30 keys max
        plan: 'pro',
        variant: 'long-form',
        'cart-size': 3,         // numbers coerced to strings
        'is-returning': true,   // booleans too
    },
});
```

Guard the call in case the script has not loaded yet (ad blockers):

```javascript
window.wireboardEvent?.({ category: 'user', action: 'login' });
```

Returns `true` if accepted, `false` if rejected (missing required fields, no Publisher ID resolvable, or session cap reached). `true` means the browser queued the event, not that it was counted: see "Verify" in [`SKILL.md`](SKILL.md). In most code paths fire-and-forget is fine.

`events.min.js` defines `window.wireboardEvent` itself and holds events until `wireboard.js` is ready, so the load order of the two scripts does not break it; loading it after the snippet is still the recommended order.

### React: click handler

```jsx
function CTAButton() {
    const handleClick = () => {
        window.wireboardEvent?.({
            category: 'engagement',
            action: 'cta-click',
            label: 'hero',
        });
    };
    return <button onClick={handleClick}>Get started</button>;
}
```

### React: visibility tracking

`IntersectionObserver` is not in React natively. Use [`react-intersection-observer`](https://www.npmjs.com/package/react-intersection-observer):

```jsx
import { useInView } from 'react-intersection-observer';
import { useEffect } from 'react';

function PromoBanner() {
    const { ref, inView } = useInView({ threshold: 0.5, triggerOnce: true });

    useEffect(() => {
        if (inView) {
            window.wireboardEvent?.({
                category: 'marketing',
                action: 'promo-view',
                label: 'summer-sale',
            });
        }
    }, [inView]);

    return <div ref={ref} className="promo-banner">Summer Sale</div>;
}
```

`triggerOnce: true` is the key flag. Without it the event fires every time the user scrolls the element back into view.

### React: form submit

Track in the success branch, not on submit, so failed submissions don't pollute conversion counts:

```jsx
function SignupForm() {
    const onSubmit = async (e) => {
        e.preventDefault();
        const formData = new FormData(e.currentTarget);
        const result = await fetch('/api/signup', { method: 'POST', body: formData });
        if (result.ok) {
            window.wireboardEvent?.({
                category: 'signup',
                action: 'complete',
                props: { plan: 'pro', source: 'pricing-page' },
            });
        }
    };
    return (
        <form onSubmit={onSubmit}>
            {/* form fields */}
        </form>
    );
}
```

### Vue 3

```vue
<script setup>
function track() {
    window.wireboardEvent?.({
        category: 'engagement',
        action: 'cta-click',
        label: 'hero',
    });
}
</script>

<template>
    <button @click="track">Get started</button>
</template>
```

### Svelte 5

Svelte 5 uses `onclick` (the older `on:click` directive is deprecated). Match the project's Svelte version.

```svelte
<script>
function track() {
    window.wireboardEvent?.({
        category: 'engagement',
        action: 'cta-click',
        label: 'hero',
    });
}
</script>

<button onclick={track}>Get started</button>
```

### TypeScript

Declare the globals once (e.g. in `src/types/wireboard.d.ts`):

```ts
declare global {
    interface WireboardEventInput {
        category: string;
        action: string;
        label?: string;
        value?: string | number;
        utm?: boolean;
        publisher?: string;
        props?: Record<string, string | number | boolean>;
    }
    interface Window {
        wireboard?: (...args: unknown[]) => void;                       // the tracking script's queue/dispatch function
        wireboardEvent?: (input: WireboardEventInput) => boolean;       // fire a custom event
        wireboardSetPublisher?: (publisher: string) => boolean;         // set publisher at runtime (overrides <meta>)
        wireboardClearPublisher?: () => void;                           // clear the runtime publisher
        wireboardCleanup?: () => void;                                  // global reset (NOT a per-component cleanup)
        wireboardEnableDataLayerListener?: (opts?: {                    // opt-in GTM/dataLayer mirror
            skip?: string[];
            denyParams?: string[];
        }) => void;
    }
}
export {};
```

## SPA pitfall: dynamic re-mount fires events again

This is THE most common SPA mistake. The events script reacts to elements present in the DOM. Each remount produces a fresh DOM element with no prior tracking state, so:

- `data-wireboard-event="view"` fires **once per element instance**. If a React component is unmounted and re-mounted (modal open/close, route change), a new instance fires a new view event.
- `data-wireboard-event="load"` fires every time an element with that attribute appears in the DOM, including re-mounts.

**Rule for SPAs**: do NOT use `data-wireboard-event` for components that mount and unmount as part of normal user flow. Use `window.wireboardEvent?.()` from inside event handlers (`onClick`, `onSubmit`) and lifecycle hooks (`useEffect` gated on a meaningful state change) instead.

`window.wireboardCleanup()` exists but it is a **global reset** (disconnects all observers, clears all view-tracking markers across the whole page). It is NOT a per-component cleanup hook. Don't call it from a React `useEffect` return; you will reset tracking for unrelated components.

## GTM / dataLayer mirror (optional)

If the user already pushes events to GTM/GA4 via `window.dataLayer`, they can opt into automatic mirroring:

```javascript
window.wireboardEnableDataLayerListener();
```

Call once after `events.min.js` loads. From that point, every `window.dataLayer.push({...})` is observed and mirrored into `window.wireboardEvent({...})`. The original `dataLayer.push` is called first so GTM and other listeners keep working; the wrap is idempotent (calling enable twice is safe); any events already in `dataLayer` when you enable get replayed.

The mirror handles three push shapes:

| Shape | Example | Mapping |
| ----- | ------- | ------- |
| GA4 standard event | `dataLayer.push({event: 'purchase', value: 49.99, currency: 'EUR', items: [...]})` | Mapped to curated `{category, action, label, value, props}` for 28 GA4 events: `purchase`, `refund`, `add_to_cart`, `remove_from_cart`, `view_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`, `view_item`, `view_item_list`, `select_item`, `add_to_wishlist`, `sign_up`, `login`, `share`, `search`, `generate_lead`, `contact`, `subscribe`, `scroll`, `click` (recorded as `engagement` / `outbound_click`), `view_search_results`, `video_start`, `video_progress`, `video_complete`, `file_download`, `form_start`, `form_submit`. |
| UA-style legacy | `dataLayer.push({event: 'name', eventCategory: 'x', eventAction: 'y', eventLabel: 'z', eventValue: 1})` | Pass-through to the same `category`/`action`/`label`/`value`. Remaining fields become props. |
| Unknown event | `dataLayer.push({event: 'my_custom_name', foo: 'bar'})` | `category: 'custom'`, `action: <eventName>`, remaining fields become props |

**For dataLayer pass-through only**, PII-flavored keys are auto-stripped from props before send: `email`, `user_email`, `email_address`, `phone`, `phone_number`, `user_phone`, `mobile`, `first_name`, `last_name`, `name`, `full_name`, `user_name`, `address`, `street`, `city`, `state`, `zip`, `postal_code`, `ssn`, `social_security`, `credit_card`, `card_number`, `cvv`, `cvc`, `password`, plus every key starting with `gtm.`. `user_id` is deliberately kept (GA4 uses it across devices): if the site pushes a `user_id` that identifies a person, add it to `denyParams` (below). GTM-internal events (`gtm.js`, `gtm.dom`, `gtm.load`, `gtm.click`, `gtm.linkClick`, `gtm.formSubmit`, `gtm.scrollDepth`, `gtm.video`, `gtm.timer`, `gtm.triggerGroup`, `gtm.elementVisibility`, `gtm.historyChange`, `gtm.init`, `gtm.init_consent`, `optimize.*`, `consent`) and `page_view` (the WireBoard tracker already counts pageviews) are skipped. Objects and arrays in custom params are dropped from props (only scalars pass through).

> **This auto-strip applies ONLY to the dataLayer mirror path.** Direct `wireboardEvent({props: {...}})` calls and `data-wireboard-event-prop-*` attributes are NOT auto-stripped. PII handling is your responsibility on those paths.

To extend the skip or deny lists:

```javascript
window.wireboardEnableDataLayerListener({
    skip: ['my_internal_event', 'debug_ping'],
    denyParams: ['internal_user_id', 'session_id']
});
```

The mirror is opt-in and off by default.
