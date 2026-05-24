<p align="center">
  <a href="https://wireboard.io">
    <img src="https://wireboard.io/img/logo-blue.png" alt="WireBoard" height="64">
  </a>
</p>

<h1 align="center"><code>wireboard</code> plugin for Claude Code</h1>

<p align="center">
  Install <a href="https://wireboard.io">WireBoard</a> analytics in your codebase from inside <a href="https://claude.com/claude-code">Claude Code</a>.
</p>

<p align="center">
  Tracking script, SPA route tracking, custom events, cookie consent flow, and a built-in debugger.<br>
  Claude does the wiring, framework-aware, no copy-pasting from docs.
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="license: MIT"></a>
  <a href="https://code.claude.com/docs/en/plugins"><img src="https://img.shields.io/badge/Claude%20Code-plugin-7c5cff.svg" alt="Claude Code plugin"></a>
  <a href="https://wireboard.io/docs"><img src="https://img.shields.io/badge/docs-wireboard.io-1f8fff.svg" alt="docs: wireboard.io"></a>
</p>

---

## What it does

Tells Claude how to install and debug WireBoard analytics in any web codebase, the right way for your framework, without you copy-pasting from docs:

- **Tracking script install** in any framework's `<head>`: Next.js (App + Pages Router), Nuxt 3, SvelteKit, Remix, Astro, Vite (React/Vue), Laravel Blade, Rails, Django, WordPress, plain HTML, Wix/Squarespace
- **SPA client-side route tracking** wired correctly per router: `usePathname`, `routeChangeComplete`, `useLocation`, `afterEach`, `afterNavigate`, `astro:after-swap`, and the vanilla History API
- **Custom events**, declarative (`data-wireboard-event-*` attributes) or programmatic (`window.wireboardEvent({...})`), with framework-specific patterns for React click handlers, visibility tracking, form submits, Vue, Svelte, and TypeScript types
- **Cookie consent flow**: load cookieless, upgrade on accept (`upgradeStorage`), revoke on withdraw. Works with Cookiebot, OneTrust, Termly, Osano, or any custom banner
- **Debugging**: when something breaks, Claude walks you through a diagnostic flow (network filter by collector hostname, console checks, manual event firing) instead of guessing
- **Built-in guardrails**: never fabricates appId/UUID values, never inspects tracker source, never leaks internal architecture, follows the WireBoard API contract documented in the skill

## Install

In Claude Code, add the [Anthropic community marketplace](https://github.com/anthropics/claude-plugins-community) (if you haven't already) and install:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install wireboard@claude-community
```

The first command is a one-time setup that gives you access to every plugin in the community catalog; the second installs WireBoard specifically.

<details>
<summary>Or install directly from this repo (faster updates, useful before the community catalog has synced the latest commit)</summary>

```
/plugin marketplace add wireboard/wireboard-claude-plugin
/plugin install wireboard@wireboard
```

Both `wireboard` parts in `wireboard@wireboard` are intentional: the first is the plugin name, the second is the marketplace name (this repo doubles as its own single-plugin marketplace).

</details>

## Usage

Three ways to invoke it, all load the same content:

**Natural language** (recommended). Just talk to Claude:

> Install WireBoard on this Next.js site.

> Add a click tracker on the "Buy Now" button. Category `ecommerce`, action `purchase`, value `49.99`.

> Add a view tracker on the hero section without mount loops.

> The dashboard shows one pageview per session on my React Router app, fix it.

> I have a Cookiebot banner, set up wireboard with the consent flow.

> The events aren't showing up in my dashboard, debug it.

**Short slash command**:

```
/install
```

(Shows as `/install (WireBoard)` in your autocomplete.)

**Fully qualified slash command**:

```
/wireboard:install
```

Use this form when you want to disambiguate from any other `/install` skills you might have. Plugin skills are namespaced as `/<plugin>:<skill>`.

## What you need from your WireBoard dashboard

Before you start, have these ready:

1. **Your tracking snippet**. Open the site manager directly: [wireboard.io/dashboard?action=site-edit](https://wireboard.io/dashboard?action=site-edit), select your site, click **Install**, copy the script block. Paste it into your chat with Claude when asked. (The long way: [wireboard.io/dashboard](https://wireboard.io/dashboard), click **Manage your Sites** in the sidebar, then proceed as above.)
2. **Your Publisher UUID**. Found in [Settings, Integrations](https://wireboard.io/dashboard/settings/integrations). Also embedded in the snippet above, so the plugin will extract it from what you paste.

Claude will NEVER invent these values; it always asks for your snippet first.

Don't have a WireBoard account yet? Sign up at [wireboard.io/register](https://wireboard.io/register). Free plan available.

## What this plugin does NOT cover

- **WireBoard public REST and Live (SSE) APIs**. For programmatic access to your data, use the official SDKs:
    - JavaScript / TypeScript: [`@wireboard/api`](https://www.npmjs.com/package/@wireboard/api)
    - Python: [`wireboard-api`](https://pypi.org/project/wireboard-api/)
    - LLM agents (MCP): [`@wireboard/mcp`](https://www.npmjs.com/package/@wireboard/mcp)
- **General analytics work** unrelated to WireBoard.

## License

MIT. See [LICENSE](LICENSE).
