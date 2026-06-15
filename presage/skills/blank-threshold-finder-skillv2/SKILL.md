---
name: presage-blank-thresholds
description: >
  Use this skill whenever the user asks about missing, unconfigured, or blank thresholds
  (spec limits) for analyses in Presage LIMS. Triggers include: "which analyses are missing
  thresholds", "blank thresholds", "no spec limits configured", "analyses without limits",
  "which tests have no thresholds", "threshold coverage", "which options have no spec",
  "compare threshold coverage across plants", "which plant is missing specs",
  "unconfigured analyses", "tests with no pass/fail criteria", "find analyses without
  choices configured", "where are thresholds not set up", or any request to audit whether
  spec limits are properly configured. This skill explains where thresholds are stored
  (AnalysisOptionChoice), what "blank" means, the correct query pattern, and how to
  compare threshold coverage across plants.
---

# Presage Blank Thresholds Skill

## Why This Skill Exists

Thresholds (spec limits) in Presage are stored as `choices` on each `AnalysisOption`.
An option with an empty `choices` array has **no threshold configured** — results entered
for that option will not be evaluated against any spec. Finding and auditing these gaps
is a distinct workflow.

---

## Key Concept: Where Thresholds Live

```
Analysis
  └── options[]               ← AnalysisOption (what to measure)
        └── choices[]         ← AnalysisOptionChoice (the threshold records)
```

Each `AnalysisOptionChoice` defines one rule:
- `choice`       — the threshold value (as a string)
- `constraint`   — `MAXIMUM` | `MINIMUM` | `NOT_EQUAL` | `NONE`
- `boundaryType` — `UNACCEPTABLE` | `MARGINAL` | `BOUNDARY` | `ALLOWED`
- `plant`        — scoped to a specific plant (null = applies to all)
- `product`      — scoped to a specific product (null = applies to all)
- `severityClass`— scoped to a severity class (null = applies to all)
- `productBatch` — scoped to a specific batch (null = applies to all)
- `active`       — whether the threshold is currently active

A **blank threshold** = an option where `choices` is an empty array `[]`, or where
all choices have `active: false`.

---

## Threshold Patterns by valueType

Any valueType can have thresholds. The *form* they take differs:

| valueType  | Typical threshold pattern |
|------------|--------------------------|
| `NUMBER`   | MINIMUM / MAXIMUM with UNACCEPTABLE or MARGINAL boundary |
| `CHOICE`   | `NONE` constraint + ALLOWED or UNACCEPTABLE per valid choice value |
| `TEXT`     | `NOT_EQUAL` or `NONE` constraint matching specific string values |
| `BOOLEAN`  | `NONE` constraint + ALLOWED / UNACCEPTABLE for true/false |
| `DATETIME` | Less common; may use MINIMUM / MAXIMUM for date range bounds |

Do **not** assume any valueType is exempt from thresholds — TEXT, BOOLEAN, and DATETIME
options can all have meaningful pass/fail criteria depending on the analysis.

---

## Step 0: Session Setup

1. `get_session_info` — check if already logged in
2. If not: `get_plants` → `authenticate` with any available plantId
3. `plantId` in `authenticate` is audit context only — it does not restrict which plants
   you can query. Pass all target plant IDs in `plantIds` filters.

---

## Core Query: Fetch Analyses with Option Choices

```graphql
query {
  analyses(
    filter: {
      plantIds: [<one or more plant IDs>]
      activeOnly: true
    }
    pagination: { pageNumber: 1, pageSize: 50 }
  ) {
    data {
      id
      name
      category
      inUseAtPlants { id name code }
      options {
        option
        valueType
        unit
        choices {
          id
          active
          choice
          constraint
          boundaryType
          plant { id name code }
          product { id name }
          severityClass { id name }
        }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

Paginate through all pages if `info.totalPages > 1`. Combine all `data` arrays before
filtering.

---

## Identifying Blank Thresholds (Client-Side)

After fetching, for each analysis → option:

```
blank threshold = choices is empty ([])
               OR all choices have active: false
```

Group findings by:
- `analysis.name` + `option.option` — the specific parameter missing a spec
- `analysis.category` — to help prioritize (micro vs. chemistry vs. calibration)
- `inUseAtPlants` — which plants run this analysis and would be affected

---

## Comparing Threshold Coverage Across Plants

Thresholds can be **globally configured** (plant: null) or **plant-scoped** (plant: <id>).

To check coverage for a target plant X:
- **Covered** if at least one active choice has `plant === null` (global) OR `plant.id === X`
- **Not covered** if all active choices reference other specific plants, with no global fallback

**Critical:** the presence of *any* plant-scoped choices does **not** imply global coverage.
If an option has choices only for Plants A and B, Plant C is uncovered — even though
thresholds exist on the option. The absence of a `plant: null` entry means every
unnamed plant has a gap.

Present cross-plant results as a table:
- Rows: analysis + option
- Columns: each plant in scope
- Cells: ✅ covered (global or plant-specific choice exists) / ❌ not covered

---

## Scoping Nuance: Product & Severity Class

The same strict rule applies to `product` and `severityClass`:
- A choice scoped to Product A does **not** cover Product B or unspecified products
- A choice scoped to Severity Class X does **not** cover other severity classes
- Only a choice where `product === null` provides coverage for all products on that option
- Only a choice where `severityClass === null` provides coverage for all severity classes

**An option can have many choices and still have gaps** if those choices are all
narrowly scoped. When auditing, always check: is there at least one active choice
that covers the target plant/product/severityClass combination, either by matching
it specifically or by having null (global) in that dimension?

**Example:** an option has exactly two choices:
- `{plant: Plant A, product: Product 1}`
- `{plant: Plant B, product: Product 2}`

The gaps this creates:
- Plant A + Product 2 → ❌ not covered (Plant A's only choice is Product 1)
- Plant B + Product 1 → ❌ not covered (Plant B's only choice is Product 2)
- Product 1 at any plant other than Plant A → ❌ not covered
- Product 2 at any plant other than Plant B → ❌ not covered
- Any other plant + any product → ❌ not covered

Each scope dimension (plant, product, severityClass) is evaluated independently and
strictly. A match on one dimension never implies coverage on another.

---

## Product Inheritance: Ancestor Thresholds Count as Coverage

A product **inherits** thresholds from its ancestors. If a choice is scoped to a
parent (or grandparent, or any ancestor) product, that choice covers all of that
product's descendants.

The `Product` type exposes this hierarchy via:
- `parentProductId` — the immediate parent's ID
- `parentProduct { id name parentProductId parentProduct { ... } }` — recursive object
  allowing traversal up the tree in a single query

**Coverage rule with inheritance:**
A product P is covered for a given option if there exists at least one active choice
where the choice's `product` is null (global), equals P, or equals any ancestor of P
— all the way up to the root.

**How to resolve inheritance when auditing:**

When you have a target product, fetch its full ancestor chain before checking coverage:

```graphql
product(id: <id>) {
  id name
  parentProduct {
    id name
    parentProduct {
      id name
      parentProduct {
        id name
        parentProduct { id name }
      }
    }
  }
}
```

Then collect the set: `[product.id, parent.id, grandparent.id, ...]`

A choice covers the target product if its `productId` is null OR is in that ancestor set.

**Practical note:** Product hierarchies in Presage are typically shallow (2–4 levels),
but the depth is not fixed. When in doubt, traverse until `parentProduct` is null.

**Important:** inheritance is product-only. Plant and severityClass scoping does
**not** inherit — a choice scoped to one plant covers only that exact plant.

---

## Presenting Results

**Single-plant audit:**
> "Found N options with no active thresholds across M analyses. Here are the gaps by category..."

**Cross-plant comparison:**
Use a table. Highlight analyses that are inconsistently configured (covered at some plants
but not others) — these are often accidental gaps rather than intentional omissions, and
worth flagging for follow-up.

---

## Useful Filter Options

- `nameContains: "pH"` — scope audit to analyses matching a keyword (partial, case-insensitive)
- `activeOnly: false` — include discontinued analyses (usually not needed for gap audits)
- Large installs may have many analyses — use pageSize: 50 and paginate fully before reporting
