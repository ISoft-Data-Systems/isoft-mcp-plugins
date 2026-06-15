# ITrack Enterprise Claude Skills

Skills for use with the ITrack Enterprise MCP connector in Claude Desktop.

## Prerequisites

1. **Claude Desktop** installed
2. **ITrack Enterprise MCP** installed and pointed to your server's GraphQL endpoint
(e.g. `https://your-company.itrackenterprise.com/graphql`)

## Installing

**Install as a plugin (recommended):** Add this marketplace and enable the
`enterprise-mcp` plugin. Both paths work on the free plan — see the
[repository README](../README.md#install) for step-by-step instructions:

- **claude.ai (web or Desktop):** Customize → Plugins → **+** → Create plugin → Add
  marketplace → `https://github.com/isoftdata/isoft-mcp-plugins`, then enable
  `enterprise-mcp`.
- **Claude Code (CLI):** `/plugin marketplace add isoftdata/isoft-mcp-plugins` then
  `/plugin install enterprise-mcp@isoft`.

**Upload a single skill:** In claude.ai, go to **Customize → Skills** and upload either
a skill's `SKILL.md` (from its folder under [skills/](skills/)) or a `.zip` of the skill
folder (use the zip when the skill has a `references/` folder).

## Available Skills

|Skill|Description|
|-|-|
|`itrack-enterprise-business-logic`|Business logic, data structure, and reporting knowledge for sales, purchasing, work orders, labor, inventory, locations, cores, and more|
|`itrack-enterprise-help`|Routes ITrack Enterprise how-to questions to the correct help documentation|
|`itrack-enterprise-query`|Best practices for querying the ITrack Enterprise GraphQL API|
|`itrack-enterprise-mysql`|Direct MySQL schema reference and query patterns for ITrack Enterprise database tables. Requires admin-level access and a separate agreement. ⚠️|
|`itrack-enterprise-custom-fields`|Querying Custom Fields (Q\&A fields) on customer, vendor, inventory, and vehicle records|
|`itrack-enterprise-web-links`|Generates deep links to records and views in ITrack Enterprise Web Edition ⚠️|

> ⚠️ **itrack-enterprise-mysql** requires admin-level database access and a separate user agreement covering proprietary schema access. Not available to all users.

> ⚠️ **itrack-enterprise-web-links** is blocked by [EEW-586 — Support deep links](https://isoftdata.atlassian.net/browse/EEW-586). Deep links only work for users already authenticated in their browser session until this is resolved.

