# WireBoard Analytics MCP for Claude Code

Ask Claude about your [WireBoard](https://wireboard.io) analytics in plain language. This plugin connects Claude Code to the official, open-source [WireBoard MCP server](https://github.com/wireboard/mcp): read-only access to your sites, period totals, daily history, time series, breakdowns by country, device, browser, language, referrer or page, top URLs, custom events with their campaigns, and the live snapshot of who is on your site right now.

## Install

In Claude Code:

```text
/plugin marketplace add wireboard/wireboard-claude-plugin
/plugin install wireboard-mcp@wireboard
```

Claude Code asks for your **WireBoard API token**. Create one at [wireboard.io/dashboard/settings/api](https://wireboard.io/dashboard/settings/api) with the `analytics:read` ability (add `live:read` for the live snapshot). Claude Code stores it in your system's secure storage. To change it later: `/plugin`, **Installed** tab, **WireBoard Analytics MCP**, **Configure options**, then `/reload-plugins`.

Requires Node.js 18 or later (the server runs with `npx`). To also let Claude install and debug WireBoard tracking in your code, install the `wireboard` plugin from the same marketplace.

## Example prompts

> How many visitors did my site get last week, compared with the week before?

> What's happening on my site right now?

> Which countries and referrers brought the most visitors this month?

> How many purchase events came from utm_source=newsletter yesterday?

## What the plugin runs, and where data goes

The plugin starts `@wireboard/mcp`, pinned to an exact version, on your computer with `npx` (downloaded from the npm registry). The server sends read-only requests, authenticated with your token, to the WireBoard API at `https://api.wireboard.io`, only when Claude calls one of its tools. It changes nothing in your account and sends nothing else anywhere: no telemetry, no conversation content. Without a token, every tool answers with instructions to add one.

You can switch the server off in `/mcp`, or remove the plugin with `/plugin`. The live snapshot needs a paid WireBoard plan; on the free plan, history queries cover the last 30 days.

## Documentation and support

- MCP server documentation: [wireboard.io/docs/api-mcp](https://wireboard.io/docs/api-mcp)
- Support: [wireboard.io/contact](https://wireboard.io/contact) or [GitHub issues](https://github.com/wireboard/wireboard-claude-plugin/issues)
- [Privacy policy](https://wireboard.io/privacy) and [terms of service](https://wireboard.io/terms)

## License

MIT. See [LICENSE](https://github.com/wireboard/wireboard-claude-plugin/blob/master/LICENSE).
