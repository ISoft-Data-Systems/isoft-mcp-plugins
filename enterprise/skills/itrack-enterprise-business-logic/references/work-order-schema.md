# Work Order Schema Reference

## Data Structure

**Work Order header** (`WorkOrder` type):

| GraphQL field | Notes |
|---|---|
| `WorkOrder.workOrderId` | |
| `WorkOrder.customer` | |
| `WorkOrder.customerUnit` | |
| `WorkOrder.type` | WO type — check flags, not name |
| `WorkOrder.date` | **Document date — used by built-in reports for filtering. Not dateCreated.** |
| `WorkOrder.dateCreated` | System creation timestamp — different set of WOs from date |
| `WorkOrder.dateClosed` | Finalization date |
| `WorkOrder.datePromised` | Promised completion date |
| `WorkOrder.closed`, `WorkOrder.void` | |
| `WorkOrder.price`, `WorkOrder.tax` | |

**Job/line** — introspect `Job` type for full field list:

| GraphQL field | Notes |
|---|---|
| *(introspect Job type)* | `billingHours`, `expectedHours` — flag hours and flat-rate estimate |
| *(introspect Job type)* | `complaint`, `cause`, `correction` — 3C fields |
| *(introspect Job type)* | `laborRate`, `laborCharge`, `fixedLabor`, `useLaborAtCost`, `useLaborHours` — labor billing |
| `timeStarted` | *(not in GraphQL API)* — when work began; used by Work Order Labor Charges report as date anchor |
| `timeEntered` | *(not in GraphQL API)* — when the job record was created |
| `timeFinished` | *(not in GraphQL API)* — when the job was marked complete |

**Time entries** (`workclock` table) — not in GraphQL API. See `references/labor-time-clock-queries.md`.

**Parts used** — available via `WorkOrder.jobs → Job.parts`.

## Date Anchor Reference

Different labor-related queries use different date anchors. When a user asks for data "for a date range" without specifying which anchor, clarify or default to clock-in time (most common for labor queries):

| Use case | Date anchor | GraphQL field | Notes |
|---|---|---|---|
| Labor Summary By Employee (clock-based) | Clock-in time | *(not in GraphQL API)* | Clock entries that occurred in the window — MySQL skill required |
| Work Order Labor Charges | Job start time | *(not in GraphQL API)* | Jobs where work began in the window — open WOs included if start falls in range — MySQL skill required |
| WO closed in period | Finalization date | `WorkOrder.dateClosed` | WOs finalized in the window |
| WO created in period | Creation timestamp | `WorkOrder.dateCreated` | System creation timestamp |
| WO document date | Document date | `WorkOrder.date` | User-editable — used by some built-in WO reports |

## Work Order Type Flags

> ⚠️ **`workOrderTypes` is not a valid GraphQL query** — attempts to query it via GraphQL fail schema validation. Work order type flags are not currently queryable via the GraphQL API. If the MySQL skill is available, query the `workordertype` table directly (see `work-orders.md`). Otherwise, ask the user to confirm which WO type flags apply to their setup, or note the limitation.

Names are user-configurable and meaningless — always use the flags, not the name.

## Time Active Calculation

> Not available via GraphQL API. If the MySQL skill is available, time active is calculated as `TIMESTAMPDIFF(MINUTE, wo.datecreated, IFNULL(wo.dateclosed, NOW())) / 60.0`. Use `MINUTE/60.0` not `HOUR` — the latter truncates and won't match report figures. Timestamps are stored in UTC; apply `CONVERT_TZ` to match report output. See `work-orders.md`.

## Worker Groups

`job.workergroupid` links to the `workergroup` table. Worker group data is not available via the GraphQL API — `WorkerGroup` does not exist as a GraphQL type. The GraphQL `Group` type is a separate permission/access system, not the same thing.

> Worker group queries are not possible via GraphQL. If the MySQL skill is available, query `workergroup` and `workergroupvalue` tables (see `work-orders.md`). Otherwise, note the limitation to the user.

## Average Labor Hours and Average $/Hour

> Not available via GraphQL API — requires actual clocked hours from `workclock`. If the MySQL skill is available, see `labor.md` — "Average Labor Hours and $/Hour" section.

**Concepts (apply regardless of query method):**
- Report averages use **actual clocked hours** (not `job.billinghours`) as the numerator
- **All WOs are in the denominator** — including zero-hour WOs. Do not average only over WOs with clock entries.
- Average $/hour = `avg_subtotal ÷ avg_actual_clocked_hours` — an aggregate metric, not a per-WO rate
