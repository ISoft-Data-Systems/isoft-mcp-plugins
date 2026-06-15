# ITrack Enterprise Claude Skills

Skills for use with the ITrack Enterprise MCP connector in Claude Desktop.

## Prerequisites

1. **Claude Desktop** installed
2. **ITrack Enterprise MCP** installed and pointed to your server's GraphQL endpoint
(e.g. `https://your-company.itrackenterprise.com/graphql`)

## Installing Skills

1. Retrieve the `SKILL.md` files from GitHub under `/ai-skills/tree/main/ISoft/skills`
2. In Claude → **Customize** → **+** → **Create a Skill** → **Upload skill from file**

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

