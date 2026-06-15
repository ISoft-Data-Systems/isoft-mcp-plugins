---
name: itrack-enterprise-query
description: "Best practices for querying the ITrack Enterprise GraphQL API and SQL via evaluateSelectQuery using the Enterprise-Api-Extension MCP. Always use this skill whenever fetching any list of entities — inventory, customers, sales orders, vehicles, vendors, work orders, purchase orders, or any other collection. Correct field names, filter syntax, orderBy format, SQL restrictions, and markdown contamination rules must all be verified against this skill before writing any query. When the query involves customer or vendor Custom Fields (Q&A fields), also use the custom-fields skill. ALSO load this skill immediately whenever a query returns any of these errors: 'Query is not a valid SELECT query', 'GRAPHQL_VALIDATION_FAILED', 'Unknown column', 'ER_BAD_FIELD_ERROR', 'not a SELECT query', or any GraphQL schema validation error — the Common Mistakes table in this skill likely contains the fix."
metadata:
  author: ISoft Data Systems, Inc.
  version: 0.3.0
  mcp-server: Enterprise-Api-Extension
  category: universal
---

# Enterprise API Best Practices

---

## Authentication

### Required sequence
1. Call `get_stores_for_login` to get available store IDs (this works without auth)
2. Call `authenticate` with a `selectedStoreId`
   - If it succeeds → session is now active, proceed with queries
   - If it returns `MCP error -32602: Tool authenticate disabled` → a session is already active, proceed with queries normally
3. Proceed with queries — the session persists until `close_session` is called

> ⚠️ **`-32602: Tool authenticate disabled` is not a failure.** It means the tool is already logged in. Do not treat this as an error — just continue to `query_graphql`.

---

## Query Rules

### Critical: Variables Must Be a JSON Object, Not a String

The `query_graphql` tool accepts a `variables` parameter. **Always pass it as a plain JSON object — never as a JSON-encoded string.**

```
# WRONG — variables double-encoded as a string (causes Bad Request error)
variables: "{\"pagination\": {\"pageNumber\": 1, \"pageSize\": 100}}"

# CORRECT — variables as a plain object
variables: {"pagination": {"pageNumber": 1, "pageSize": 100}}
```

The API will return this error if you pass variables as a string:

> `variables` in a POST body should be provided as an object, not a recursively JSON-encoded string.

This applies to all queries that use GraphQL variables (`$pagination`, `$filter`, etc.).
As an alternative, you can inline the values directly into the query string to avoid variables entirely:

```graphql
# Inline — no variables parameter needed
query {
  customers(pagination: { pageNumber: 1, pageSize: 100 }) { ... }
}
```

---

## SQL via `evaluateSelectQuery`

### Availability

GraphQL (`query_graphql`) is available to all authenticated users. SQL via `evaluateSelectQuery` requires elevated permissions. If it fails with a permissions error: use GraphQL where the data is available via a supported query type, or direct the user to the relevant screen in the ITrack application UI. Always prefer GraphQL for entities that have a supported query type; fall back to `evaluateSelectQuery` when GraphQL doesn't cover the data needed.

### Required Sub-fields

`evaluateSelectQuery` returns a `QueryEvaluation` object — always select `{ success result message }`:

```graphql
{ evaluateSelectQuery(query: "SELECT ...") { success result message } }
```

Writing `{ evaluateSelectQuery(query: "...") }` without sub-fields fails schema validation.

### MySQL Syntax

`evaluateSelectQuery` uses **MySQL syntax**, not SQL Server:
- Row limiting: `LIMIT N` — not `TOP N`
- Date math: `TIMESTAMPDIFF`, `DATE_SUB`, `DATE_ADD`, `DATE()`
- Standard: `JOIN`, `GROUP BY`, `ORDER BY`, `HAVING`, `CASE WHEN`
- CTEs (`WITH ... AS`) and subqueries in `FROM` are **not supported** — see restrictions below

### Timestamps and Timezone

The DB stores timestamps in UTC. Built-in reports render in the server's local timezone — SQL queries against raw timestamps will differ from printed report figures by the UTC offset. This affects any duration calculation (`TIMESTAMPDIFF` on clock entries, time active, etc.) and can cause ~1hr variance per record. For cross-DST-boundary records, exact reproduction is ambiguous.

To match report output: check `@@session.time_zone` first, then apply `CONVERT_TZ(timestamp, '+00:00', @@session.time_zone)`. If `@@session.time_zone` returns `'SYSTEM'`, ask the user for their timezone or accept the UTC difference as a known limitation.

Full details and examples are in the `itrack-enterprise-business-logic` skill.

### Restrictions

> ⚠️ **Subqueries in `FROM` are not supported**: `FROM (SELECT ...) alias` returns "Query is not a SELECT query." Restructure using JOINs, or break into separate queries and pass ID lists between them.

> ⚠️ **CTEs (`WITH ... AS`) are not supported**: Same error as above. Use inline subqueries in `JOIN` clauses instead, or split into multiple queries.

> ⚠️ **Only `SELECT` statements are allowed**: `INSERT`, `UPDATE`, `DELETE`, `CREATE`, etc. will be rejected.

### "Query is not a valid SELECT query" Error

This error almost always means a **wrong column name or table name** — not a syntax restriction. Do not assume the query structure is invalid. Check:
1. Column names against the relevant schema reference
2. Table names — e.g. `useraccount` not `user`, `workclock` not `timeclock`
3. Whether the query uses a disallowed form (subquery in FROM, CTE, non-SELECT statement)

### Markdown Link Contamination — CRITICAL

Column references with dot notation like `ua.name` or `wg.name` must be written as **plain SQL identifiers** — never as markdown links. Some environments auto-convert dotted text into hyperlinks (`[ua.name](http://ua.name)`), which silently corrupts the SQL string and produces "not a valid SELECT query." Always verify dotted column references are plain text before submitting.

### Performance

- `workclock` has no index on `intime` — date-range filters on `workclock.intime` cause full table scans. Drive queries from `workorder` or scope `workclock` via `userid IN (...)` instead.
- Unscoped `GROUP BY jobid` on `workclock` (all-time, no filter) will timeout on large datasets. Always scope to a specific jobid list first.
- The `workclock → job → workorder` multi-table join with a date filter is expensive — use the workorder-first pattern documented in `itrack-enterprise-business-logic` reference files for labor queries.

---

## GraphQL Queries

### Pre-built Tools vs. Raw GraphQL

Several MCP tools handle pagination internally. Prefer these when they cover your use case:

| Tool | Internal pageSize | Notes |
|---|---|---|
| `analyze_vehicle_portfolio` | configurable (default 500, max 1000) | Handles multi-page internally |
| `get_reorder_recommendations` | configurable (default 75, max 200) | Single-page by design |
| `analyze_top_customers` | configurable (default 10, max 100) | Single-page by design |
| `analyze_inventory_asset_value` | configurable (default 1000, max 5000) | May need multiple calls for huge datasets |

Use `query_graphql` directly only when these tools don't cover your need.

### Verified Filter Syntax

Filter field names differ per entity. Use these verified shapes:

#### Inventory
```graphql
filter: {
  statuses: [A, H]          # InventorySearchStatus enum list — A, H, S, C, D, T, B
  stores: [1]               # list of store IDs (NOT storeId: {...})
  description: "bumper"     # string wildcard match
  partNumber: "12345"
  skuNumber: 99001
  replenishable: true
}
```

#### Customers
```graphql
filter: {
  active: true              # Boolean (NOT active: { eq: true })
  companyName: "Acme"       # string wildcard
  keyword: "smith"
  ids: [456]                # list of customer IDs
}
```

#### Customer Address Coverage

Customer records have **two distinct address locations**:
- **Billing address** — top-level fields on the customer record (`address`, `city`, `state`, `zip`, `country`)
- **Alternate addresses** — stored in a nested `addresses` collection

> ⚠️ **Any query that filters or searches by address must check both locations.** Querying only `addresses` (alt addresses) will silently miss customers whose relevant address is their billing address, and vice versa.

Fetch both in the same query, then match across both in code:

```graphql
customers(pagination: { pageNumber: 1, pageSize: 100 }) {
  items {
    id
    companyName
    address   # billing address
    city
    state
    zip
    country
    addresses {   # alternate addresses
      items {
        address
        city
        state
        zip
        country
      }
    }
  }
}
```

#### Sales Orders
```graphql
filter: {
  storeId: [1]              # list of store IDs
  date: { gte: "2026-01-01", lte: "2026-02-17" }   # DateFilter — filters by CREATION date
  finalized: true
  isSale: true              # finalized, not voided, affected inventory
  customerFilter: {         # nested CustomerFilter — NOT customerId: { eq: ... }
    ids: [456]
  }
}
```

> ⚠️ **`date` filters by creation date, not `dateFinalized`.** There is no server-side filter for finalization date on Sales Orders. Use `finalized: true` to exclude unfinalized orders, and accept that `date` scopes by creation date. **Do NOT** widen the date range to compensate — this balloons the result set and forces post-fetch filtering in code.

**Anti-pattern example:** The goal is "February 2026 revenue for Store 2." The temptation is to widen to December–February to catch SOs created earlier but finalized in February:

```graphql
# WRONG — fetches 964 items across 10 pages just to filter in code
salesOrders(filter: { date: { gte: "2025-12-01", lte: "2026-02-28" }, storeId: [2] })

# ALSO WRONG — fetches 338 January-created SOs to find the ~10 finalized in Feb
salesOrders(filter: { date: { gte: "2026-01-01", lte: "2026-01-31" }, storeId: [2] })

# RIGHT — filter by February creation date; use finalized: true to exclude open orders
salesOrders(filter: { date: { gte: "2026-02-01", lte: "2026-02-28" }, storeId: [2], finalized: true })
```

If the business requirement is strictly "finalized in February regardless of creation date," that cannot be satisfied server-side. Document this limitation and work with creation-date scoping instead.

#### Vehicles
```graphql
filter: {
  stores: [2]               # list of store IDs
  statuses: ["A", "H"]      # string list — "A", "H", "S", "C", "D", "T", "B"
  make: "Ford"              # wildcard string
  year: { gte: 2015 }       # IntFilter
}
```

> ⚠️ **`model` is an object type, not a scalar.** Query it as `model { name }` — using `model` as a plain field will cause a schema validation error. Use `explore_schema` on the `Vehicle` type to see all available subfields on `model`.

#### Purchase Orders
```graphql
filter: {
  storeId: [1]              # list of store IDs
  date: { gte: "2026-01-01", lte: "2026-01-31" }   # DateFilter — filters by CREATION date
  doneReceiving: true
  pricesFinalized: true
  void: false
}
```

> ⚠️ **`date` filters by creation date.** There is no server-side filter for received date. Use `doneReceiving: true` to scope to completed POs. Do not widen the date range to catch POs received in a given period.

#### Work Orders
```graphql
filter: {
  store: { id: [2] }    # StoreFilter — NOT storeId: [2] (that only works on SOs and POs)
  date: { gte: "2026-01-01", lte: "2026-01-31" }        # DateFilter — filters by CREATION date
  dateClosed: { gte: "2026-01-01", lte: "2026-01-31" }  # DateFilter — filters by CLOSE date
  dateCreated: { gte: "2026-01-01", lte: "2026-01-31" } # DateFilter — alias for creation date
  closed: true
  isEstimate: false
  void: false
}
```

> ✅ **Work Orders expose separate date filters.** Unlike other document types, you can filter directly by `dateClosed` — no need to widen the date range. Use `dateClosed` when you want WOs closed in a specific period, and `date`/`dateCreated` when you want WOs created in a period.

#### Transfer Orders
```graphql
filter: {
  date: { gte: "2026-01-01", lte: "2026-01-31" }   # DateFilter — filters by CREATION date
  doneReceiving: true
  void: false
}
```

> ⚠️ **`date` filters by creation date.** There is no server-side filter for received date. Use `doneReceiving: true` to scope to completed transfers. Do not widen the date range to catch transfers received in a given period.

#### orderBy syntax — string enums (most entities)

Most entities use a **string enum** for `orderBy`, not an `OrderByInstruction` object. Each entity has its own enum type with `_ASC` / `_DESC` suffixed values:

| Entity | Enum type | Example values |
|---|---|---|
| Inventory | `InventorySort` | `description_ASC`, `partNumber_DESC`, `quantity_ASC` |
| Sales Orders | `SalesOrderSort` | `date_ASC`, `date_DESC` (only two values) |
| Vehicles | `VehicleSort` | `stockNumber_ASC`, `make_ASC`, `year_DESC` |
| Work Orders | `WorkOrderSort` | `workOrderId_ASC`, `workOrderId_DESC` (only two values) |
| Purchase Orders | `PurchaseOrderSort` | `date_ASC`, `date_DESC` (only two values) |

```graphql
orderBy: [date_ASC]
orderBy: [description_DESC]
orderBy: [stockNumber_ASC]
```

Use `explore_schema` with `type: enum` and the entity's sort enum name to see all valid values before writing a query.

> ⚠️ Using `{ field: "date", direction: ASC }` on any of these entities will throw: `Enum "XSort" cannot represent non-enum value`.

#### orderBy syntax — customers (exception)

Customers are the exception: they use `OrderByInstruction` object syntax, not a string enum.

```graphql
orderBy: [{ field: "companyName", direction: ASC }]
orderBy: [{ field: "dateEntered", direction: DESC }]
```

### Common Mistakes

| Mistake | Fix |
|---|---|
| Passing `variables` as a JSON-encoded string | Pass as a plain object: `{"pageNumber": 1}` not `"{\"pageNumber\": 1}"` |
| Omitting `pageInfo` | Always include it — you can't know if there are more pages without it |
| Using `totalCount` instead of `totalItems` | **The field is `totalItems`** — `totalCount` does not exist and will error |
| Assuming page 1 has all data | Always check `totalPages > 1` |
| Using `pageSize: 10000` | Cap at 1000; use iteration instead |
| No `orderBy` across pages | Add a stable sort field |
| Filtering in code after full fetch | Filter in the GraphQL query |
| `storeId: { eq: 1 }` on inventory/vehicles | Use `stores: [1]` (list of IDs) |
| `customerFilter: { ids: [456] }` on salesOrders | Known backend bug — throws `ER_BAD_FIELD_ERROR`. Workaround: get SO IDs via SQL first, then query by `SalesOrderKey` |
| `SalesOrderFilter.ids: [43685]` (int or string scalar) | `ids` takes `SalesOrderKey` objects: `ids: [{ storeId: 2, salesOrderId: 43685 }]` |
| `SalesOrderFilter.date: { from: "...", to: "..." }` | `DateFilter` uses comparison operators: `date: { gte: "2026-04-10", lte: "2026-04-10" }` |
| `salesOrder { lines { id type } }` | `lines` is paginated: `lines(pagination: { pageNumber: 1, pageSize: 100 }) { totalItems items { ... } }` |
| `active: { eq: true }` on customers | Use `active: true` (plain Boolean) |
| `status: ACTIVE` on inventory | Use `statuses: [A]` (enum list) |
| `orderBy: [{ field: COMPANY_NAME }]` on vehicles | Vehicles use string enum: `orderBy: [companyName_ASC]` |
| Querying only `addresses` for address-based lookups | Billing address is on top-level customer fields (`address`, `city`, `state`, `zip`, `country`). Always check both the billing fields and the `addresses` collection. |
| `model` on vehicles as a plain field | `model` is an object type — use `model { name }` |
| `orderBy: [{ field: "date", direction: ASC }]` on salesOrders, inventory, workOrders, purchaseOrders | Most entities use string enums — `orderBy: [date_ASC]`, `orderBy: [description_DESC]`, etc. Only customers use `OrderByInstruction` object syntax. Check the entity's sort enum with `explore_schema` before writing a query. |
| Widening date range to catch records by a different date field | The `date` filter on all document types (SOs, POs, Transfer Orders) is **creation date**. Don't expand the range — use status filters (`finalized`, `doneReceiving`) to narrow results, or use `dateClosed` on Work Orders which uniquely supports it. |
| `storeId: [2]` on workOrders | Work Orders use `store: StoreFilter`, not `storeId`. Use `store: { id: [2] }` instead. Only Sales Orders and Purchase Orders accept `storeId` directly. |
| Subqueries in `FROM` clause with `evaluateSelectQuery` | `FROM (SELECT ...) alias` returns "Query is not a SELECT query." Use JOINs instead, or break into separate queries passing ID lists between them. |
| `customer.companyname` or `customer.companyName` in SQL | The SQL column is `customer.company` — not `companyname`. GraphQL uses `companyName`; SQL uses `company`. Wrong name returns "Query is not a valid SELECT query." |
| `salesorder.closed = 1` or `salesorder.void = 0` in SQL | These are string fields — use `= 'True'` and `= 'False'`, not integers or booleans. |

---

## Pagination

### How Pagination Works

All list queries in the Enterprise API use page-number-based pagination via `PaginationOptions`:

```graphql
input PaginationOptions {
  pageNumber: Int! = 1   # 1-indexed
  pageSize:   Int! = 100 # default 100
}
```

Every list response returns a `pageInfo` object and `totalItems` (not `totalCount` — that field does not exist):

```graphql
type PageInfo {
  pageNumber: Int!   # current page (1-indexed)
  pageSize:   Int!   # items per page requested
  totalPages: Int!   # total number of pages
}
```

Pattern:
```graphql
inventories(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: { ... }
  orderBy: [{ field: "skuNumber", direction: ASC }]
) {
  items { ... }
  totalItems
  pageInfo { pageNumber pageSize totalPages }
}
```

### Core Pagination Rules

#### 1. Always request `totalItems` and `pageInfo`
Never omit these fields. They're required to know if there are more pages.

```graphql
# Always include
totalItems
pageInfo {
  pageNumber
  pageSize
  totalPages
}
```

#### 2. Choose page size based on use case

| Use Case | Recommended `pageSize` |
|---|---|
| UI display / interactive browsing | 20–50 |
| Background data processing | 100–500 |
| Full dataset retrieval (use with caution) | 500–1000 |
| Pre-built analysis tools (e.g., `analyze_vehicle_portfolio`) | Use the tool's own `pageSize` param |

The default is 100. Do not use very large page sizes (>1000) without a clear need — it stresses the server and increases response size.

#### 3. Iterate until all pages are fetched

When you need the complete dataset:

```
page 1 → check pageInfo.totalPages → if totalPages > 1, fetch pages 2..totalPages
```

Stop condition: `pageInfo.pageNumber >= pageInfo.totalPages`  
Or equivalently: `items fetched so far >= totalItems`

**Do not assume one page is enough.** Even with `pageSize: 1000`, large datasets (e.g., inventory) can span many pages.

#### 4. Use filters to reduce result sets before paginating

Always apply the most restrictive filters possible first. Fetching unfiltered lists is slow and wasteful.

```graphql
# Bad: paginate everything, then filter in code
inventories(pagination: { pageNumber: 1, pageSize: 1000 }) { ... }

# Good: filter first, then paginate
inventories(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: { statuses: [A, H], stores: [1] }
) { ... }
```

#### 5. Use `orderBy` for consistent pagination

Without a stable sort, results across pages can shift if data changes mid-fetch. Always specify `orderBy` when iterating multiple pages.

```graphql
orderBy: [{ field: "skuNumber", direction: ASC }]
```

### Full Multi-Page Retrieval Pattern

When you need all records, loop like this (pseudocode):

```
allItems = []
page = 1

loop:
  result = query(pageNumber: page, pageSize: 100)
  allItems += result.items
  
  if page >= result.pageInfo.totalPages:
    break
  page += 1
```

In practice via `query_graphql`, execute each page call separately and accumulate results. Inform the user how many total items exist and your progress.

### Count-Only Queries (pageSize: 1)

> ⚠️ **Field name**: The count field is `totalItems`, not `totalCount`. Using `totalCount` will cause a schema error.

When you only need a count, not the full records, set `pageSize: 1`. The API always returns `totalItems` for the full filtered set regardless of page size — so you get the count with minimal data transfer:

```graphql
query CountAvailableVehicles {
  vehicles(
    pagination: { pageNumber: 1, pageSize: 1 }
    filter: {
      stores: [2]
      statuses: ["A", "H"]
    }
  ) {
    totalItems
    pageInfo { pageNumber totalPages }
  }
}
```

You don't even need to request any `items` fields. Use this pattern any time the goal is a count or existence check.

### Nested Pagination

The API supports fetching nested paginated lists in a single query (e.g., customers with their salesOrders). However, note:

- Nested pagination uses the same `PaginationOptions` input
- Inner list pagination is independent of the outer list pagination
- If nested lists are large, prefer separate queries per parent entity to avoid response size explosion and n+1 patterns

```graphql
# OK for small nested lists
customers(pagination: { pageNumber: 1, pageSize: 50 }) {
  items {
    id
    companyName
    salesOrders(pagination: { pageNumber: 1, pageSize: 10 }) {
      items { salesOrderId total }
      totalItems
      pageInfo { totalPages }
    }
  }
}

# Prefer separate queries when nested lists can be large
# (e.g., a customer with thousands of orders)
```

### Deep Nesting Performance Warning

Fetching deeply nested relationships (e.g., `salesOrder → lines → job → workOrder`) across a large outer page size causes severe response time degradation — observed at **7+ seconds** for 100 SOs with line items.

This happens because the server must resolve every nested relationship for every item on the page simultaneously. The more levels deep and the more line items per SO, the worse it gets.

**Rules for deep nested queries:**

| Nesting depth | Recommended outer `pageSize` |
|---|---|
| 1 level (e.g. SO → lines) | 50–100 |
| 2 levels (e.g. SO → lines → job) | 25–50 |
| 3+ levels (e.g. SO → lines → job → workOrder) | 10–25 |

**Response times are not consistent across pages.** The cost scales with the number of nested items on each page, not just the outer page size. A page with mostly single-line SOs may take 7 seconds; a page with many multi-line SOs may take 25 seconds. This means you cannot assume a "safe" page size will be safe for every page in the dataset.

**Observed in production** — SO → lines → job → workOrder at pageSize 100:
- Pages 1–3: ~7.7–7.9 seconds each
- Page 4 (denser line items): **24.9 seconds**

**Anti-pattern example:**

```graphql
# SLOW AND UNPREDICTABLE — 7–25+ second response times at pageSize 100
salesOrders(pagination: { pageNumber: 1, pageSize: 100 }) {
  items {
    id
    subtotal
    lines {
      items {
        job {
          workOrder { id }
        }
      }
    }
  }
}

# BETTER — smaller pageSize gives more predictable, consistently fast responses
salesOrders(pagination: { pageNumber: 1, pageSize: 25 }) {
  items {
    id
    subtotal
    lines {
      items {
        job {
          workOrder { id }
        }
      }
    }
  }
}
```

If you need to process all records, accept that more pages are required and iterate — smaller pages with predictable response times are faster overall than risking a 25-second stall or timeout.

---

## Examples

### Fetch All Active Inventory from a Store

```graphql
query GetActiveInventory($page: Int!) {
  inventories(
    pagination: { pageNumber: $page, pageSize: 100 }
    filter: {
      statuses: [A, H]
      stores: [1]
    }
    orderBy: [{ field: "skuNumber", direction: ASC }]
  ) {
    items {
      id
      skuNumber
      description
      quantity
    }
    totalItems
    pageInfo {
      pageNumber
      pageSize
      totalPages
    }
  }
}
```

1. Run with `$page = 1`
2. Read `pageInfo.totalPages` — inventory datasets can be large, don't assume one page is enough
3. Loop from page 2 to `totalPages` if needed
4. Accumulate `items` from all pages

### Fetch All Active Customers

```graphql
query GetCustomers($page: Int!) {
  customers(
    pagination: { pageNumber: $page, pageSize: 100 }
    filter: { active: true }
    orderBy: [{ field: "companyName", direction: ASC }]
  ) {
    items {
      id
      companyName
    }
    totalItems
    pageInfo {
      pageNumber
      pageSize
      totalPages
    }
  }
}
```

1. Run with `$page = 1`
2. Read `pageInfo.totalPages`
3. Loop from page 2 to `totalPages` if needed
4. Accumulate `items` from all pages
