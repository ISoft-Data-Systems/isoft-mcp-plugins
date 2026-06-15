---
name: presage-product-testing
description: >
  Use this skill whenever the user asks about test results for a specific product brand, SKU,
  or product name in Presage LIMS. Triggers include any named product or brand in a test result
  request: "[brand] test results", "[brand] water testing at [plant]", "what [brand] tests were
  run", "is [product] still active", "which plants run [brand] tests", "[product] out of spec
  results", "grab sample results for [product]", "finished product testing for [brand]", "find
  samples for [product name]", or "lot/batch results for [product]". Also use when the user
  wants to know which plants carry or test a given product. The product→sample relationship is
  non-obvious — this skill prevents the most common failure (filtering samples directly by
  product name, which the API does not support).
  metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage Product Testing Skill

## Why This Skill Exists

Brand or SKU-specific test queries fail nearly every time without this pattern because product
names live on **Product** entities, not on Analyses or Samples. There is no `productName`
filter on `SampleFilter` — you must first resolve the product ID, then cross-reference samples
by plant, date range, and analysis type.

---

## The Data Model

```
Product (e.g., a specific SKU or brand)
  └── ProductBatch (lot numbers, expiration tracking)
       └── Sample (individual test instance)
            └── SampleValue (individual result)
```

The practical implication: to find samples for a product, find the product first, then
look for samples at the same plant in the relevant date range using analysis name as the
bridge.

---

## Step 0: Session Setup

1. `get_session_info` — if logged in, proceed; otherwise `get_plants` → `authenticate`

---

## Step 1: Find the Product

```graphql
query {
  products(
    filter: {
      plantIds: [<id>]
      nameContains: "<brand or SKU keyword>"
      activeOnly: false        # include discontinued — needed for historical queries
    }
    pagination: { pageNumber: 1, pageSize: 30 }
    orderBy: [NAME_ASC]
  ) {
    data {
      id
      name
      category
      productType              # INGREDIENT or FINISHED_PRODUCT
      active                   # true = currently active
      inUseAtPlants { id name code }
    }
    info { totalItems totalPages }
  }
}
```

`nameContains` does a partial, case-insensitive match on the product name.
Use the most distinctive word or phrase from the brand/SKU the user mentioned.
If multiple products match, present the list and ask the user to confirm.

---

## Step 2: Check Active Status (if that's the question)

To answer "Is [product] still active?" or "Which plants carry [product]?":

- `active: true` on a product result means currently active at that plant
- `inUseAtPlants` shows all plants where the product is configured
- Set `activeOnly: false` to include discontinued products so historical data isn't missed

---

## Step 3: Find Samples Associated with the Product

After confirming the product exists, query samples for the same plant(s) and date range.
Filter client-side by `analysis.name` to the relevant test type (e.g., "Finished Product
Testing", "Grab Sample", "Water Quality"):

```graphql
query {
  samples(
    filter: {
      plantIds: [<id or ids from inUseAtPlants>]
      performedFrom: "<from>"
      performedTo: "<to>"
    }
    pagination: { pageNumber: 1, pageSize: 120 }
    orderBy: [performed_DESC]
  ) {
    data {
      id tagNumber performed
      plant { id name code }
      analysis { id name category }
      workOrder { id title }
      sampleValues {
        result resultStatus filledOut
        analysisOption { option unit }
      }
    }
    info { totalItems totalPages }
  }
}
```

Then filter client-side: look for samples where the work order title or analysis name
contains the relevant test type the user cares about.

---

## Cross-Plant Product Discovery

"Which plants test [product]?" or "Have any facilities run [product] testing?"

1. Search `products` with `nameContains` — check `inUseAtPlants` on results
2. This tells you configuration; for actual completed tests, follow up with a sample
   query at those plant IDs filtered to the expected analysis name

---

## Batch / Lot Tracking

When the user asks about a specific lot or batch number:

```graphql
products(
  filter: { plantIds: [<id>], nameContains: "<product>" }
  pagination: { pageNumber: 1, pageSize: 10 }
) {
  data {
    id name
    productBatches {
      id
      lotNumber
      status
      expirationDate
    }
  }
  info { totalItems }
}
```

Batch `status` values: `ACTIVE`, `QUARANTINE`, `RELEASED`, `REJECTED`, `EXPIRED`

---

## Key Gotchas

❌ **Don't filter samples by product name** — `SampleFilter` has no such field
❌ **Don't search analyses for brand names** — analysis names are generic
✅ **Do use `products` query first** to resolve the product ID and plant coverage
✅ **Do use `inUseAtPlants`** — fastest answer to "which plants carry this product?"
✅ **Do set `activeOnly: false`** for historical queries — discontinued products have history

---

## Reference

See `references/queries.md` for additional product query templates including batch queries
and cross-plant product coverage.
