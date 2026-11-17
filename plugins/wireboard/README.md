# WireBoard Analytics MCP for Claude Code

Ask Claude about your website traffic, and let it install [WireBoard](https://wireboard.io) analytics for you.

WireBoard is a real-time web analytics platform: live visitors, custom dashboards, custom events, funnels and journeys, with first-party cookies or a fully cookieless mode. This plugin gives Claude Code two things:

- **Your analytics in plain language.** Ask "how many visitors did we get last week?" or "which pages under /blog bring the most traffic?" and Claude answers from your WireBoard data: period totals, daily history, time series, breakdowns by country, device, browser, referrer or page, top URLs, custom events with their UTM sources, and the live snapshot of who is on your site right now. Read-only.
- **A correct install in any codebase.** Claude adds the WireBoard tracking script in the right place for your framework, counts client-side navigation in single-page apps, instruments custom events, sets up a cookie consent flow, offers the official Google Tag Manager template when the site uses GTM, and walks you through debugging when the dashboard looks wrong.

## Install

In Claude Code:

```text
/plugin marketplace add wireboard/wireboard-claude-plugin
/plugin install wireboard@wireboard
```

Claude Code then asks for an optional **WireBoard API token**:

- Want Claude to read your analytics? Create a token at [wireboard.io/dashboard/settings/api](https://wireboard.io/dashboard/settings/api) with the `analytics:read` ability (add `live:read` for the live snapshot) and paste it. It is stored in your system's secure storage, not in a settings file.
- Only want Claude to install tracking? Leave it empty. You can add it later: `/plugin`, **Installed** tab, **WireBoard**, **Configure options**.

The analytics tools run with Node.js 18 or later (`npx`). No WireBoard account yet? [Sign up](https://wireboard.io/register), there is a free plan.

## Example prompts

> How many visitors did my site get last week, compared with the week before?

> What's happening on my site right now?

> Which countries and referrers brought the most visitors this month?

> How many purchase events came from utm_source=newsletter yesterday?

> Install WireBoard on this Next.js site.

> Add a click event on the "Buy now" button: category ecommerce, action purchase, value 49.99.

> I have a Cookiebot banner. Set up WireBoard so it only uses cookies after consent.

> My React Router app shows one pageview per visit. Fix it.

> The events aren't showing up in my dashboard. Debug it.

The install workflow is also available as `/wireboard:install`.

## What the plugin runs, and where data goes

- **Analytics tools (MCP server).** The plugin starts the official open-source [`@wireboard/mcp`](https://github.com/wireboard/mcp) server, version pinned, on your machine with `npx` (downloaded from the npm registry). It sends read-only requests, authenticated with your token, to the WireBoard API at `https://api.wireboard.io`, and only when Claude calls one of its tools. It changes nothing in your account and sends nothing else anywhere: no telemetry, no conversation content. Without a token it starts but every tool answers with instructions to add one.
- **Install skill.** Instructions only, no code runs. When you ask for an install, Claude edits your project to add the WireBoard tracking script, loaded from `https://static.wireboard.io/`. Once you deploy it, your site's visitors send pageviews and events to your WireBoard collector. What the tracker stores in visitors' browsers in each mode is documented in [Cookies and storage](https://wireboard.io/docs/cookies).

You can turn the analytics tools off on their own in `/mcp`, or remove the whole plugin with `/plugin`.

## Documentation and support

- Plugin documentation: [wireboard.io/docs/claude-plugin](https://wireboard.io/docs/claude-plugin)
- Troubleshooting a WireBoard install: [wireboard.io/docs/troubleshooting](https://wireboard.io/docs/troubleshooting)
- API and MCP server: [wireboard.io/docs/api-mcp](https://wireboard.io/docs/api-mcp)
- Support: [wireboard.io/contact](https://wireboard.io/contact) or [GitHub issues](https://github.com/wireboard/wireboard-claude-plugin/issues)
- [Privacy policy](https://wireboard.io/privacy) and [terms of service](https://wireboard.io/terms)

## License

MIT. See [LICENSE](https://github.com/wireboard/wireboard-claude-plugin/blob/master/LICENSE).
