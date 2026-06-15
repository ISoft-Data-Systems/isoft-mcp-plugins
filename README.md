# ISoft MCP Plugins

Claude Code plugins that add product-specific skills for ISoft Data Systems MCP servers.

Each plugin is scoped to one product. **Install only the plugin for the product you use** — you don't need the others.

## Install

Add the marketplace once:

```
/plugin marketplace add isoftdata/isoft-mcp-plugins
```

Then install the plugin for your product:

| Product | Command |
| --- | --- |
| Presage Analytics | `/plugin install presage-mcp@isoft` |
| ITrack Enterprise | `/plugin install enterprise-mcp@isoft` |

## Plugins

- **[presage-mcp](presage/)** — skills for the Presage Analytics MCP server.
- **[enterprise-mcp](enterprise/)** — skills for the ITrack Enterprise MCP server.

## For ISoft employees

Adding the marketplace lets you browse and install any product's plugin for testing:

```
/plugin marketplace add isoftdata/isoft-mcp-plugins
/plugin install presage-mcp@isoft
/plugin install enterprise-mcp@isoft
```
