---
name: presage-out-of-spec
description: >
  Use this skill whenever the user asks about out-of-spec, failed, warning, or at-risk test
  results in Presage LIMS. Triggers include: "out of spec results", "failed tests", "which plants
  have the most issues", "at-risk results", "food safety risk", "OOB results", "warnings",
  "non-conforming samples", or any request to find, count, rank, or summarize results that are
  not passing. Also use when comparing plants by quality performance, identifying problem trends,
  or finding which analyses have the highest failure rates. This skill provides the exact query
  patterns and result status values needed — always read it before building out-of-spec queries.
  metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage Out-of-Spec Results Skill

## Why This Skill Exists

Out-of-spec queries follow a consistent pattern that's easy to get wrong without the right
field names and enum values. This skill gives you everything you need upfront so you don't
have to discover it through schema exploration.

---

## Step 0: Session Setup

1. Call `get_session_info` — if already logged in, proceed
2. If not logged in: call `get_plants` to get available plant IDs and codes, ask the user
   which plant(s) to authenticate against if unclear, then call `authenticate`
3. The `plantId` used for authentication is for session context only — it does not restrict
   which plants you can query. Pass any `plantId` from `get_plants`.

---

## The ResultStatus Enum — Know These Before Querying

```
OUT_OF_BOUNDS   → Hard fail: result exceeds defined limits
ERROR           → System or calculation error on result
WARNING         → Soft fail: result outside advisory range
ALLOWED         → Passing result
NOT_CALCULATED  → Result exists but no threshold evaluation applied
```

"Out of spec" = `OUT_OF_BOUNDS` + `ERROR`
"At risk" = add `WARNING`
"Any non-passing" = everything except `ALLOWED`

---

## Core Query Pattern

```graphql
query {
  samples(
    filter: {
      plantIds: [<one or more plant IDs>]
      performedFrom: "<YYYY-MM-DDT00:00:00Z>"
      performedTo: "<YYYY-MM-DDT23:59:59Z>"
    }
    pagination: { pageNumber: 1, pageSize: 120 }
    orderBy: [performed_DESC]
  ) {
    data {
      id
      tagNumber
      performed
      plant { id name code }
      analysis { id name category }
      sampleValues {
        id
        result
        resultStatus
        filledOut
        analysisOption { option unit }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

After fetching, **filter client-side** for non-ALLOWED sampleValues. This is more efficient
than multiple narrow queries when you need full context for ranking or grouping.

---

## Multi-Plant Queries

Pass all target plant IDs in a single `plantIds` array — do NOT close and re-authenticate
per plant. One session handles all plants. Get IDs from `get_plants` if not already known.

---

## Ranking Plants by Out-of-Spec Count

1. Query all target plants in one call
2. Filter `sampleValues` to `resultStatus in [OUT_OF_BOUNDS, ERROR]` client-side
3. Group by `plant.name`, count non-ALLOWED values
4. Sort descending and present as a ranked table

---

## Risk Ranking Logic

When asked about "food safety risk" or "highest risk results", weight by analysis category.
The general principle: microbiological failures carry more food safety risk than chemical
deviations, which carry more risk than physical or calibration checks. Ask the user to
confirm their organization's risk priority if it's not clear from context.

Present findings with clear severity tiers:
- 🔴 Critical: microbiological `OUT_OF_BOUNDS` / `ERROR`
- 🟡 Important: chemistry `OUT_OF_BOUNDS` / `ERROR`
- ⚪ Monitor: `WARNING` results of any category
- 🔵 Informational: physical/calibration deviations

---

## Pagination

If `info.totalPages > 1`, increment `pageNumber` and re-query until you have all data.
Combine result arrays before grouping or ranking.

---

## Presenting Results

Lead with the summary number (total samples reviewed, total OOB+ERROR values, % failure
rate), then break down by plant → analysis category → specific analysis → individual
results (only if user asks for that level of detail). Use a table for multi-plant comparisons.

---

## Reference

See `references/queries.md` for additional ready-to-run query templates including
blank-value detection and specific-analysis filtering.
