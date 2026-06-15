---
name: presage-qc-workflow
description: >
  Use this skill whenever the user asks about operational QC workflow items in Presage —
  open or unclosed work orders, pending validations, calibrations, verifications, and incomplete
  tests. Triggers include: "open validations", "open work orders", "unclosed samples", "pending
  calibrations", "[test name] calibration completed", "open items needing completion", "Vision
  System Verification", "Scale Calibration", "pH meter calibration", "thread rinser checks",
  "batch room cleaning", "incomplete tests", "missing results", "which work orders are still
  open", "what hasn't been closed", "blank/unfilled results". This skill covers both the
  WorkOrder query path AND the get_blank_values tool, which is purpose-built for finding
  missing results and is often the fastest answer to "what's incomplete?"
  metadata:
  author: ISoft Data Systems
  version: 0.1
  mcp-server: Presage
  category: universal
---

---

# Presage QC Workflow Skill

## Why This Skill Exists

QC workflow queries — calibrations, validations, open work orders, incomplete tests — follow
a different query path than analytical sample queries. Knowing when to use `workOrders` vs.
`samples` vs. `get_blank_values` saves many exploratory tool calls.

---

## Decision Tree: Which Tool to Use?

```
User asks about...
│
├── "Incomplete / missing / unfilled / blank results"
│   └── USE: get_blank_values tool (fastest, purpose-built)
│
├── "Open / unclosed work orders" or "pending documents"
│   └── USE: workOrders query with statuses: [OPEN, SAMPLED]
│
├── "Calibration / verification / validation completed (or failed)"
│   └── USE: samples query — find the analysis first (presage-analysis-finder pattern)
│        then filter by performed date range
│
└── "Failed / out-of-spec calibration results"
    └── USE: presage-out-of-spec pattern + filter client-side by analysis name
```

---

## Tool 1: get_blank_values (Start Here for Missing Results)

The `get_blank_values` MCP tool finds unfilled sample values (where `filledOut` is null).
Use it instead of a custom query whenever the user asks what hasn't been filled in.

```
Parameters:
  plantIds: [Int]            — required
  modifiedFrom: DateTime     — optional date range start
  modifiedTo: DateTime       — optional date range end
  optionName: String         — optional: filter to a specific option/parameter name
  resultStatuses: [Enum]     — optional: filter by ResultStatus
  pageNumber: Int
  pageSize: Int (max 500)
```

**When to use:** "What results are missing?", "What hasn't been filled out?",
"What needs to be entered?", "Any incomplete test entries?"

---

## Tool 2: Work Order Query

```graphql
query {
  workOrders(
    filter: {
      plantId: <Int>                    # ⚠️ SINGULAR — not an array
      statuses: [OPEN, SAMPLED]         # omit to include all statuses
      from: "YYYY-MM-DD"               # date only, no time component
      to: "YYYY-MM-DD"
    }
    pagination: { pageNumber: 1, pageSize: 50 }
    orderBy: { columnName: "dateCreated", direction: "DESC" }
  ) {
    data {
      id
      title
      documentStatus
      scheduled
      dateCreated
      plant { id name code }
      samples {
        id
        tagNumber
        status
        analysis { id name category }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

**Critical:** `workOrders` takes `plantId` (singular integer), NOT a `plantIds` array.
For multi-plant work order queries, run the query once per plant within the same session —
no need to close and re-authenticate between calls.

**Work order status values:**
- `OPEN` — Created, samples not yet started
- `SAMPLED` — In progress, results being entered
- `CLOSED` — Complete
- `CANCELLED` — Voided

---

## Tool 3: Calibration / Verification Sample Query

For finding completed (or failed) calibration and verification tests:

1. Find the analysis using the `presage-analysis-finder` pattern with a relevant keyword
   (e.g., "Calibration", "Verification", "CAL-", "Scale", "Vision")
2. Query samples in the date range and filter client-side by `analysis.name`

```graphql
samples(
  filter: {
    plantIds: [<id>]
    performedFrom: "YYYY-MM-DDT00:00:00Z"
    performedTo: "YYYY-MM-DDT23:59:59Z"
  }
  pagination: { pageNumber: 1, pageSize: 120 }
) {
  data {
    id tagNumber performed status
    analysis { id name category }
    workOrder { id title documentStatus }
    sampleValues { result resultStatus filledOut analysisOption { option unit } }
  }
  info { totalItems totalPages }
}
```

---

## Finding Open Validations / Overdue Items

"Are there any open validations at [plant] for the past 2 weeks?"

1. `get_current_date` to calculate the 2-week window
2. Query `workOrders` with `statuses: [OPEN, SAMPLED]` and the date range
3. Optionally filter work order `title` client-side for relevant keywords
4. Report: count of open items, their age in days since `dateCreated`, which samples
   are still in OPEN vs. SAMPLED state

---

## Calibration Coverage Audit

To check whether calibrations are being completed on schedule:

1. Query all work orders (including `CLOSED`) in the period
2. Group by `title` (which typically contains the calibration name and period)
3. Flag any expected calibration periods that have no CLOSED work order
4. Report: completed, pending, and missing calibration windows

---

## Presenting QC Workflow Results

**Open work orders:** List with title, age in days, document status, and sample completion state.

**Missing results:** From `get_blank_values` — report count by plant and parameter name,
and how long they've been blank (gap between `modifiedFrom` and today).

**Calibration status:** Report whether each expected calibration period has a CLOSED sample.
Flag any with OPEN/SAMPLED status past the expected completion date.

---

## Reference

See `references/queries.md` for work order and calibration query templates, and pagination
patterns for multi-plant work order audits.
