---
name: presage-api-navigation
description: >
  Use this skill when working with the Presage LIMS (Laboratory Information Management System)
  via the presage-api-mcp MCP server. Covers authentication, schema navigation, querying work
  orders, samples, products, analyses, and avoiding context bloat from large GraphQL payloads.
  metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage API Navigation Skill

## Overview

You have access to an MCP server connected to the Presage Web GraphQL API — a multi-plant LIMS.
Your job is to answer questions about lab samples, work orders, products, and analyses by
querying this API efficiently.

> **IMPORTANT:** Only `query_graphql` is available for data retrieval. Mutations are **not permitted**.

---

## Step 1: Authentication

You **must** authenticate before most queries. Follow this exact sequence:

1. Call `get_plants` — returns a list of plants with `id`, `name`, and `code`.
2. Ask the user which plant to log into, or use context clues from their request.
3. Call `authenticate` with the chosen `plantId`.

> **Key fact:** The `plantId` used for authentication is **only for session context and audit trails**.
> It does **not** limit which plants you can query. Once authenticated, `query_graphql` can access
> data from any authorized plant using filters.

After authenticating, you may call `get_session_info` to confirm your session status.

---

## Step 2: Schema Navigation

**Never guess GraphQL field names or query structure.** Use the schema tools:

1. `search_schema(keyword: "...")` — Find relevant types by keyword (e.g. `"workOrder"`, `"sample"`, `"product"`).
2. `explore_schema(type: "type", items: "TypeName")` — Inspect the exact fields of a specific type.
3. `explore_schema(type: "input", items: "FilterName")` — Inspect filter/input types before writing queries.

Only write a query after you have confirmed the field names exist in the schema.

---

## Step 3: Querying Safely

### Context Bloat Warnings

Large, unfiltered responses **will crash your context window**. You must:

- **Always** use date range filters (`from`/`to`, `performedFrom`/`performedTo`) — never query the full history.
- **Always** use `pagination` with a reasonable `pageSize` (10–50 for most cases, up to 120 for samples).
- **Only request fields you need** — do not select every field on every nested type.
- **Never loop through all plants one by one** to aggregate data — use multi-plant filters or the `inUseAtPlants` pattern (see `references/queries.md`).

### Scratchpad Protocol

After each query, summarize findings in a `<scratchpad>` block before responding:

```
<scratchpad>
- Query: workOrders for plant 1, last 7 days
- Total found: 14 (OPEN: 5, SAMPLED: 6, CLOSED: 3)
- Top issue: 3 work orders flagged with out-of-spec samples
</scratchpad>
```

This prevents you from re-querying for information you already have.

---

## Step 4: Iterative Refinement

Start narrow and expand only as needed:

1. Run a small query first (`pageSize: 5`) to verify the response shape.
2. Confirm field names match what you expect before requesting more data.
3. Use GraphQL error messages — they often suggest correct field names.
4. Add fields incrementally; always start with `id` and `status`.

---

## Reference Files

Read these files when you need them — do not load them proactively:

| File | When to read it |
|------|----------------|
| `references/presage_api_mcp_quick_reference.md` | Entity relationships, field name gotchas, lifecycle info |
| `references/queries.md` | Pre-written, optimized query patterns for common tasks |

---

## Available Tools Summary

| Tool | Purpose |
|------|---------|
| `get_plants` | List all plants (id, name, code) |
| `authenticate` | Log in with a plantId |
| `search_schema` | Find schema types by keyword |
| `explore_schema` | Inspect fields of a specific type or input |
| `query_graphql` | Execute read-only GraphQL queries |
| `get_server_information` | Get API schema version and release number |
| `get_session_info` | Check current session/auth status |
| `quick_reference` | Load the bundled API quick reference resource |
| `get_current_date` | Get today's date in ISO format |
| `close_session` | End the current session |
