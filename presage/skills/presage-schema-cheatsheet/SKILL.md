---
name: presage-schema-cheatsheet
description: >
  Use this skill as a quick-reference supplement whenever working with the Presage LIMS GraphQL
  API. Read it before calling search_schema or explore_schema for common types — it contains the
  field names, filter structures, enum values, response shapes, and gotchas for the most
  frequently used Presage types. Triggers include: any Presage query where you would otherwise
  call search_schema for SampleFilter, SampleValue, ResultStatus, AnalysisOption, PageInfo,
  WorkOrder, Product, or Analysis fields. Also read when unsure about field naming conventions,
  response structure, or ordering enum formats. Reading this once per session typically eliminates
  the need for schema exploration tool calls on the most common query patterns.
  metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage Schema Cheat Sheet

Read this once at the start of a Presage session. It covers the types you'll use in nearly
every query, so you don't have to rediscover them through schema tools.

---

## Session & Authentication Pattern

```
1. get_session_info        → check if already logged in
2. (if not) get_plants     → retrieve available plant IDs and codes
3. authenticate            → any plantId works; it's for audit context only,
                             not a restriction on what you can query
4. query with plantIds: [all target IDs] in filters
5. close_session           → when fully done
```

The `plantId` in `authenticate` does NOT limit which plants are accessible.
All plants you have permission for can be queried in a single session.

---

## SampleFilter (samples query)

```graphql
filter: {
  plantIds: [Int]            # array — pass multiple plants in one call
  performedFrom: DateTime    # "YYYY-MM-DDT00:00:00Z"
  performedTo: DateTime      # "YYYY-MM-DDT23:59:59Z"
  statuses: [SampleStatus]   # OPEN, SAMPLED, CLOSED, CANCELLED
}
# ⚠️ No direct product name filter on SampleFilter — resolve via products query first
```

---

## SampleValue Fields

```graphql
sampleValues {
  id
  result          # String — cast to float for numeric analysis
  resultStatus    # ResultStatus enum
  filledOut       # DateTime — null means the result was never entered
  analysisOption {
    option        # ⚠️ NOT "name" — the field is "option"
    unit
    valueType     # ⚠️ NOT "dataType"
  }
}
```

---

## ResultStatus Enum (complete)

| Value | Meaning |
|-------|---------|
| `OUT_OF_BOUNDS` | Hard fail — exceeds defined limits |
| `ERROR` | Calculation or system error |
| `WARNING` | Advisory range exceeded |
| `ALLOWED` | Passing |
| `NOT_CALCULATED` | No threshold applied |

---

## Response / Pagination Shape

All paginated queries use this structure — no exceptions:

```graphql
{
  data: [...]          # ⚠️ NOT "items"
  info: {
    totalItems: Int    # ⚠️ NOT "totalRecords"
    totalPages: Int
    pageNumber: Int
    pageSize: Int
  }
}
```

---

## Sample Ordering Enums

```graphql
orderBy: [performed_DESC]    # ✅
orderBy: [tagNumber_ASC]     # ✅
orderBy: [TAG_NUMBER_ASC]    # ❌ wrong format
```

---

## WorkOrder Filter

```graphql
filter: {
  plantId: Int               # ⚠️ SINGULAR — workOrders does NOT accept plantIds array
  statuses: [DocumentStatus] # OPEN, SAMPLED, CLOSED, CANCELLED
  from: "YYYY-MM-DD"         # date only — no time component
  to: "YYYY-MM-DD"
}
orderBy: { columnName: "dateCreated", direction: "DESC" }
```

For multi-plant work order queries: run one query per plant in the same session.
Do NOT close and re-authenticate between plants.

---

## Analysis Filter & Fields

```graphql
analyses(
  filter: {
    plantIds: [Int]           # array supported
    nameContains: String      # partial, case-insensitive
    activeOnly: Boolean       # false = include inactive/discontinued
  }
) {
  data {
    id name category
    analysisType              # TESTING or RECIPE
    inUseAtPlants { id name code }
    options {
      option                  # ⚠️ NOT "name"
      valueType               # ⚠️ NOT "dataType"
      unit
      requiredToClose
      requiredToPerform
      defaultValue
    }
  }
}
```

---

## Product Filter & Fields

```graphql
products(
  filter: {
    plantIds: [Int]
    nameContains: String      # partial, case-insensitive — use brand keyword here
    activeOnly: Boolean       # false = include discontinued
  }
) {
  data {
    id name category
    productType               # INGREDIENT or FINISHED_PRODUCT
    active
    inUseAtPlants { id name code }
    productBatches { id lotNumber status expirationDate }
  }
}
```

---

## Common Field Name Gotchas

| ❌ Wrong | ✅ Right | Context |
|---------|---------|---------|
| `name` | `option` | AnalysisOption |
| `dataType` | `valueType` | AnalysisOption |
| `acceptability` | `resultStatus` | SampleValue |
| `items` | `data` | All response types |
| `totalRecords` | `totalItems` | PageInfo |
| `TAG_NUMBER_ASC` | `tagNumber_ASC` | Sample ordering |
| `plantIds: [Int]` | `plantId: Int` | WorkOrder filter only |

---

## DateTime Format Reference

```
Sample queries:   "YYYY-MM-DDT00:00:00Z"  (start of day)
                  "YYYY-MM-DDT23:59:59Z"  (end of day)
WorkOrder filter: "YYYY-MM-DD"            (date only, no time)
```

---

## get_blank_values Tool (Often Overlooked)

A dedicated MCP tool for finding unfilled/missing sample results — use it instead of
writing a custom query when the user asks about incomplete, blank, or missing test data.

Parameters: `plantIds` (required), `modifiedFrom`, `modifiedTo`, `optionName`,
`resultStatuses`, `pageNumber`, `pageSize` (max 500).

Returns SampleValue records where `filledOut` is null.
