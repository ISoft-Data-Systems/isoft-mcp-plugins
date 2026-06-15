# ISoft MCP Plugins

Claude Code plugins that add product-specific skills for ISoft Data Systems MCP servers.

Each plugin is scoped to one product. **Install only the plugin for the product you use** — you don't need the others.

## Install

You can install these plugins from the claude.ai GUI or the Claude Code CLI. Both
work on the free plan.

### In claude.ai (web or Desktop)

Add the marketplace once:

1. Open **Customize → Plugins**.
2. Click **+ → Create plugin → Add marketplace**.
3. Enter the repository URL: `https://github.com/ISoft-Data-Systems/isoft-mcp-plugins`

The marketplace then appears under **Personal**. Click into it and enable the plugin
for your product (`presage-mcp` or `enterprise-mcp`) — enable only the one you use.

### In Claude Code (CLI)

Add the marketplace once:

```
/plugin marketplace add ISoft-Data-Systems/isoft-mcp-plugins
```

Then install the plugin for your product:

| Product           | Command                                |
| ----------------- | -------------------------------------- |
| Presage Analytics | `/plugin install presage-mcp@ISoft`    |
| ITrack Enterprise | `/plugin install enterprise-mcp@ISoft` |

### Without installing a plugin (single skill)

If you'd rather not add the whole plugin, you can upload one skill at a time in
claude.ai under **Customize → Skills**. Upload either the skill's `SKILL.md` on its
own, or a `.zip` of the skill's folder (use the zip when the skill has a `references/`
folder so those files come along). See each plugin's README for the list of skills and
their folder paths.

## Plugins

- **[presage-mcp](presage/)** — skills for the Presage Analytics MCP server.
- **[enterprise-mcp](enterprise/)** — skills for the ITrack Enterprise MCP server.
