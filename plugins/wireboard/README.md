# WireBoard Analytics for Claude Code

Let Claude install [WireBoard](https://wireboard.io) analytics correctly in your codebase, and debug it when the dashboard looks wrong.

WireBoard is a real-time web analytics platform: live visitors, custom dashboards, custom events, funnels and journeys, with first-party cookies or a fully cookieless mode. This plugin teaches Claude Code how to set it up the right way for your stack:

- **Tracking script in the right place** for Next.js (App and Pages Router), Nuxt, SvelteKit, React Router (including former Remix apps), Astro, Vite, Angular, Laravel (Blade and Inertia), Rails, Django, WordPress, plain HTML, Wix and Squarespace. The snippet is kept exactly as copied, so the dashboard can detect it.
- **Single-page apps counted correctly**: one pageview per real page change, with the new page's title and the previous page as referrer, for every common router, Inertia, Rails Turbo and Livewire.
- **Custom events** as HTML attributes or `window.wireboardEvent()` calls, with patterns for React, Vue, Svelte and Angular.
- **Cookie consent**: cookieless until the visitor accepts, cookies afterwards, everything erased when consent is withdrawn. Works with Cookiebot, OneTrust, Termly, Osano or your own banner.
- **Google Tag Manager**: the official WireBoard template, including its Consent Mode.
- **Debugging**: a step-by-step check of the script, the requests and their answers, the console, the bot filter and the IP Ignore List.

## Install

In Claude Code:

```text
/plugin marketplace add wireboard/wireboard-claude-plugin
/plugin install wireboard@wireboard
```

Then ask Claude, or run `/wireboard:install`. Claude asks for your tracking snippet from the [site manager](https://wireboard.io/dashboard?action=site-edit). No WireBoard account yet? [Sign up](https://wireboard.io/register), there is a free plan.

## Example prompts

> Install WireBoard on this Next.js site.

> Add a click event on the "Buy now" button: category ecommerce, action purchase, value 49.99.

> I have a Cookiebot banner. Set up WireBoard so it only uses cookies after consent.

> My React Router app shows one pageview per visit. Fix it.

> The events aren't showing up in my dashboard. Debug it.

## What the plugin runs, and where data goes

Instructions for Claude only: the plugin runs no code and sends nothing anywhere. When you ask for an install, Claude edits your project to add the WireBoard tracking script, loaded from `https://static.wireboard.io/`. Once you deploy it, your site's visitors send pageviews and events to your WireBoard collector. What the tracker stores in visitors' browsers in each mode is documented in [Cookies and storage](https://wireboard.io/docs/cookies).

Want Claude to answer questions about your traffic too? The read-only analytics tools are a separate plugin from the same marketplace, `wireboard-mcp` (see the [plugin documentation](https://wireboard.io/docs/claude-plugin)).

## Documentation and support

- Plugin documentation: [wireboard.io/docs/claude-plugin](https://wireboard.io/docs/claude-plugin)
- Installation guide: [wireboard.io/docs/installation](https://wireboard.io/docs/installation)
- Support: [wireboard.io/contact](https://wireboard.io/contact) or [GitHub issues](https://github.com/wireboard/wireboard-claude-plugin/issues)
- [Privacy policy](https://wireboard.io/privacy) and [terms of service](https://wireboard.io/terms)

## License

MIT. See [LICENSE](https://github.com/wireboard/wireboard-claude-plugin/blob/master/LICENSE).
