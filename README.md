# StyleBI Claude Code Plugins

A [Claude Code](https://claude.com/claude-code) plugin marketplace for
[StyleBI](https://github.com/inetsoft-technology/stylebi), maintained by
[InetSoft](https://www.inetsoft.com). Each plugin drives a real StyleBI deployment through natural
language via MCP tools.

## Plugins

| Plugin | What it does |
|---|---|
| [`stylebi-admin-chat`](admin/) | Administer StyleBI Enterprise Manager — server properties, schedule tasks, permissions, identities, providers, data sources, cluster nodes, and more — with a reviewable, audited, all-or-nothing apply flow. |
| [`stylebi-composer-chat`](composer/) | Edit a live StyleBI sheet open in the Composer: worksheet structure, viewsheet layout and formatting, chart and table data binding, and viewsheet JavaScript. |

Each plugin's own README (linked above) has the full detail — prerequisites, available commands,
and known limitations. `stylebi-viz-chat` (visualization creation from natural language) is not yet
published here; it will be added in a future release.

## Install

```
/plugin marketplace add inetsoft-technology/stylebi-claude-plugins
/plugin install stylebi-admin-chat@stylebi-chat
/plugin install stylebi-composer-chat@stylebi-chat
```

Install either plugin independently — you don't need both. Both run as local stdio MCP servers
with the runtime bundled in; no build step, `npm install`, or `node_modules` needed. Restart Claude
Code after installing so it loads the server.

**Updating**, once a new version is published here:

```
/plugin marketplace update stylebi-chat
/plugin update stylebi-admin-chat@stylebi-chat
/plugin update stylebi-composer-chat@stylebi-chat
```

## Requirements

- A running StyleBI deployment (Enterprise Manager reachable for admin-chat; the Composer reachable
  for composer-chat).
- A user account with the appropriate access for the plugin you're using — see that plugin's own
  README for specifics (a Site Administrator account for admin-chat; `wiz.agent.pairing.enabled`
  turned on server-side for composer-chat).

## Support

For issues with a specific plugin, open an issue in this repository. For StyleBI itself, see
[inetsoft-technology/stylebi](https://github.com/inetsoft-technology/stylebi).
