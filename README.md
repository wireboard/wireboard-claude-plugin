<p align="center">
  <a href="https://wireboard.io">
    <img src="https://wireboard.io/img/logo-blue.png" alt="WireBoard" height="64">
  </a>
</p>

# WireBoard plugins for Claude Code

Official [WireBoard](https://wireboard.io) plugins for Claude Code. This repository is a plugin marketplace with two plugins:

| Plugin | What it does | Install |
| --- | --- | --- |
| [`wireboard`](plugins/wireboard/README.md): **WireBoard Analytics** | Claude installs WireBoard tracking correctly in any codebase (framework, single-page apps, custom events, cookie consent, Google Tag Manager) and debugs it. Instructions only, no code runs. | `/plugin install wireboard@wireboard` |
| [`wireboard-mcp`](plugins/wireboard-mcp/README.md): **WireBoard Analytics MCP** | Claude answers questions about your traffic through the official read-only [WireBoard MCP server](https://github.com/wireboard/mcp). Needs a WireBoard API token and Node.js 18+. | `/plugin install wireboard-mcp@wireboard` |

Add the marketplace once, then install either or both:

```text
/plugin marketplace add wireboard/wireboard-claude-plugin
/plugin install wireboard@wireboard
/plugin install wireboard-mcp@wireboard
```

Documentation: [wireboard.io/docs/claude-plugin](https://wireboard.io/docs/claude-plugin). Support: [wireboard.io/contact](https://wireboard.io/contact) or [GitHub issues](https://github.com/wireboard/wireboard-claude-plugin/issues).

## License

MIT. See [LICENSE](LICENSE).
