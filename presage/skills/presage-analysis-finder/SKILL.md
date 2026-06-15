---
name: presage-analysis-finder
description: >
  Use this skill whenever the user asks which plants run a specific test, whether any facility
  has ever conducted a particular analysis, or wants to discover where a test type is used
  across the network. Triggers include: "which plants run X", "have any facilities recorded
  results for Y", "does [plant] record Z", "what plants perform [test name]", "which plants
  run [test]", "any location complete [analysis]", "find where [test] is used", "cross-plant
  analysis discovery". Also use when searching for analyses by brand name or product SKU,
  or by category (micro, calibration, grab sample, water testing). Always read this skill
  before building analysis discovery queries — the two-step lookup pattern here prevents
  the most common failure modes.
metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage Analysis Finder Skill

## Why This Skill Exists

Finding which plants run a specific test requires a specific two-step approach. Jumping
straight to sample queries without first resolving the analysis ID leads to empty results
or excessive schema exploration. And searching for product brand names in analysis names
will never work — brand names live on Products, not Analyses.

---

## Step 0: Session Setup

1. Call `get_session_info` — if already logged in, proceed
2. If not: `get_plants` → `authenticate` with any available plant ID

---

## The Two-Step Pattern

### Step 1: Find the Analysis

```graphql
query {
  analyses(
    filter: {
      plantIds: [<any authenticated plant ID>]
      nameContains: "<keyword>"   # partial, case-insensitive
      activeOnly: false           # cast wide net — include inactive
    }
    pagination: { pageNumber: 1, pageSize: 50 }
  ) {
    data {
      id
      name
      category
      analysisType
      inUseAtPlants { id name code }
    }
    info { totalItems totalPages }
  }
}
```

The `inUseAtPlants` field immediately tells you which plants have this analysis configured —
**this may fully answer "which plants run X?" without any further queries.**

### Step 2: Confirm with Sample Data (if needed)

`inUseAtPlants` reflects configuration, not recent activity. If the user wants to know
whether tests have *actually been run* in a given period, follow up with a sample query
filtered to those plant IDs and date range, then filter client-side by `analysis.name`.

```graphql
query {
  samples(
    filter: {
      plantIds: [<plant IDs from inUseAtPlants>]
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
      sampleValues { result resultStatus analysisOption { option unit } }
    }
    info { totalItems totalPages }
  }
}
```

---

## Choosing Keywords for nameContains

Start with the most distinctive word in the analysis name — the noun or qualifier that
makes it unique. If results are empty or too broad, adjust:

- Too many results → add a second word or qualifier
- Zero results → try a shorter or alternate term
- Try both capitalized and lowercase variants if unsure

If the user is asking about a product brand or SKU (e.g., "[brand] testing" or "[product]
water tests"), the brand name is on the **Product**, not the Analysis. Use the
`presage-product-testing` skill for those queries instead. Analysis names tend to be
generic: "Finished Product Testing", "Grab Sample", "Water Quality", etc.

---

## Cross-Plant Discovery Pattern

When the user wants to know if *any* facility runs a test:

1. Query `analyses` with `inUseAtPlants` — this is the fastest path
2. If `inUseAtPlants` is empty for an analysis, the test is not configured at any plant
3. If you get multiple matching analyses (e.g., different versions of a test), present
   all of them and let the user clarify

---

## When Analysis ID Is Already Known

If you've already resolved the analysis ID in the current session, skip Step 1 entirely
and pass `plantIds` to the sample query directly. Analysis IDs are stable within a session.

---

## Common Failure Modes to Avoid

❌ Don't search for brand/product names in `analyses` — they won't match
❌ Don't assume the auth plant is the only searchable plant — pass all target `plantIds`
❌ Don't loop through plants one by one — `plantIds` accepts an array
❌ Don't skip `inUseAtPlants` — it often answers the question without a sample query

---

## Reference

See `references/queries.md` for extended templates including category-based browsing
and multi-analysis discovery.
