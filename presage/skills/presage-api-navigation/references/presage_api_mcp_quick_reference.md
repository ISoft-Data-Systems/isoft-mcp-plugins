# Presage API MCP — Quick Reference

## System Overview

Presage Web is a multi-plant LIMS with these core domains:

- **Sample Management** — Lab sample collection, testing, result tracking
- **Work Order Management** — Document workflow for sample collection and recipe production
- **Analysis Management** — Test definitions with options, rules, and thresholds
- **Product Management** — Ingredients and finished products with batch/lot tracking
- **Plant Operations** — Multi-plant support with hierarchical locations and process zones
- **Quality Control** — Investigation triggers, severity classification, compliance tracking

---

## Entity Relationships

```
Plant
├── WorkOrder (many per plant)
│   └── Sample (many per WorkOrder)
│       └── SampleValue (many per Sample — individual test results)
├── Analysis (many per plant)
│   └── AnalysisOption (many per Analysis — defines what to test)
└── Product (many per plant)
    └── ProductBatch (many per Product — lot/batch tracking)
```

**Key relationships:**
- `WorkOrder` → `Sample`: 1:many — samples belong to a work order
- `Sample` → `SampleValue`: 1:many — each value is one test result
- `Analysis` → `AnalysisOption`: 1:many — options define individual measurements
- `Product` → `ProductBatch`: 1:many — batch tracking for lot control
- **Note:** Samples do not link directly to Products in all cases — the link may go through `ProductBatch`

---

## Common Field Name Gotchas

These are the most frequent mistakes. Check this list before writing queries.

| Wrong | Correct | Type |
|-------|---------|------|
| `SampleValue.acceptability` | `SampleValue.resultStatus` | Field name |
| `AnalysisOption.name` | `AnalysisOption.option` | Field name |
| `AnalysisOption.dataType` | `AnalysisOption.valueType` | Field name |
| `Response.items` | `Response.data` | Response shape |
| `Response.totalRecords` | `PageInfo.totalItems` | Pagination field |
| `TAG_NUMBER_ASC` | `tagNumber_ASC` | OrderBy enum (samples) |

---

## Response Shape

Almost all paginated queries return this structure:

```graphql
{
  data: [...]           # Array of results (NOT "items")
  info: {
    totalItems: Int!    # Total count (NOT "totalRecords")
    pageNumber: Int!
    pageSize: Int
    totalPages: Int!
  }
}
```

---

## Status Lifecycles

**Work Order (DocumentStatus):**
```
OPEN → SAMPLED → CLOSED → CANCELLED
```

**Sample (status):** Tracked separately from the work order's `documentStatus`.
Do not confuse `Sample.status` with `WorkOrder.documentStatus`.

---

## Schema Navigation Patterns

```graphql
# Find types related to a keyword
search_schema(keyword: "workOrder")

# List all fields on a type
explore_schema(type: "type", items: "Sample")

# Inspect a filter/input type
explore_schema(type: "input", items: "SampleFilter")

# Inspect multiple types at once
explore_schema(type: "type", items: ["Sample", "SampleValue"])
```

---

## Analysis Types

- **TESTING** — Standard lab analysis performed on samples
- **RECIPE** — Production batch / recipe analysis

---

## Field Verification Strategy

When unsure about fields:
1. Run `explore_schema(type: "type", items: "EntityName")` to see exact fields
2. Start with just `id` and `status`, add fields incrementally
3. Read GraphQL error messages — they often name the correct field
4. Test with `pageSize: 1` before requesting larger result sets
