---
name: itrack-enterprise-business-logic
description: "Use for any question involving ITrack Enterprise data, business logic, calculations, reporting, or troubleshooting. Covers sales orders, purchasing, work orders, labor efficiency, GL, AR, picking/shipping, inventory structure, locations, warehouse management, cores, and report building. Trigger on: sales figures, profit, COGS, inventory valuation, cost pools, vehicle teardown, parts on a truck, unpulled parts, work order efficiency, labor hours, purchasing, AR aging, picking, shipping, deliveries, warehouse management, inventory counts, barcode scanning, bin locations, variance, average cost, cores, document types, or how any ITrack calculation works. For Q&A custom field querying, also load itrack-enterprise-custom-fields."
metadata:
  author: ISoft Data Systems, Inc.
  version: 2.0.0
  mcp-server: Enterprise-Api-Extension
  category: universal
---

# ITrack Enterprise — Business Logic and Reporting

---

## Check for Pre-Built Reports First

Before building a custom query, check whether a relevant pre-built report or dashboard already exists.

**Crystal Reports** — must use pagination (`reports` is not a bare list query):
```graphql
{
  reports(pagination: { pageNumber: 1, pageSize: 200 }) {
    totalItems
    items { id category name description }
  }
}
```

> ⚠️ `{ reports { id category name ... } }` without pagination arguments will fail schema validation.

**Dashboard reports:**
```graphql
{ dashboardReports { id title charts { title type } } }
```

If a relevant report exists, direct the user to it. Build a custom query only if no suitable report exists, the user wants a different format, or they need different calculations than the standard report provides.

To get parameters for a specific report, use `report(id: N)` not `reportQueueParameter` — the latter queries the report queue system, not report definitions:
```graphql
{ report(id: 359) { id name description parameters { name description defaultValue } } }
```

---

## Query Approach — GraphQL First

> **Before writing any GraphQL query, read the `itrack-enterprise-query` skill file.** It has verified filter syntax, correct pagination field names (`pageNumber` not `page`, `totalItems` not `totalCount`), and performance guidance. The errors that come from guessing these are preventable.

**Reference files — read before writing queries in these domains:**
- Sales Order queries → `references/sales-order-schema.md`
- COGS / cost queries → `references/cogs-queries.md`
- GL / adjustment queries → `references/gl-categories-schema.md`
- Work Order queries → `references/work-order-schema.md`
- **Labor / time clock queries → `references/labor-time-clock-queries.md`** ← read this first; some data is not yet in the GraphQL API
- Purchasing queries → `references/purchasing-schema.md`
- Inventory queries → `references/inventory-schema.md`
- Location / warehouse queries → `references/locations-schema.md`

GraphQL (`query_graphql`) is available to all authenticated users and is the primary query method.

Some data is not yet available in the GraphQL API. Where noted, the **itrack-enterprise-mysql skill** may provide access if it is available to the user (it requires admin-level access and a separate agreement). If that skill is not available, direct the user to built-in reports or note the limitation clearly — do not fail silently.

### Always Introspect Before Writing a Query

**Never guess field names, filter names, or pagination syntax.** Always check:

```graphql
# Object fields
{ __type(name: "WorkOrder") { fields { name description type { name kind ofType { name } } } } }

# Filter fields — ALWAYS introspect separately from the object type
{ __type(name: "WorkOrderFilter") { inputFields { name type { name kind ofType { name } } } } }

# Pagination structure
{ __type(name: "PaginationOptions") { inputFields { name type { name kind ofType { name } } } } }
```

Filter field names ≠ object field names. Introspect both.

### Decision Order
1. Introspect the object type
2. Introspect the filter input type
3. Introspect pagination
4. Write the query
5. If data is not in the GraphQL API, note the limitation and mention the MySQL skill if relevant

### GraphQL Field Name Gotchas

| Gotcha | Notes |
|---|---|
| `UserAccount.name` | Login/username — NOT `username`. `firstName` and `lastName` are separate optional fields. |
| `UserAccount.id` | Primary key — not `userId`. The DB primary key is `useraccountid`, not `userid`. |
| `userAccounts` query | Query is `userAccounts`, not `users`. No pagination required — returns all accounts. |
| `Customer.companyName` | Business name. `Customer` has no `name` field — use `companyName` for the business and `contactName` for the individual. |
| Worker groups | `WorkerGroup` does not exist as a GraphQL type. If the MySQL skill is available, see `work-orders.md`. |

---

## Resolving Relative Dates

**Always call `get_current_date` before building any query with relative date language:** "today", "yesterday", "this week", "last week", "this month", "last month", "this year", "last year", "YTD", "recent", "current", "past X days."

Derive date ranges after receiving the result. Show the resolved range to the user before executing.

---

## Report Presentation Guidance

Ask about preferred presentation style before generating: Summary (totals by dimension), Detail (one row per transaction), By date (aggregated by day/week/month), or Ranked (top N by metric).

### Cross-Validation
All built-in sales reports draw from the same data. Grand totals should match across reports covering the same scope. Any discrepancy indicates a filter or scope difference.

---

## Business Logic: Sales Orders

> **Schema reference:** Read the file `references/sales-order-schema.md` before writing any SO or SOL query.

A qualifying sale (`isSale = true`) = `canaffectinventory = True` AND `closed = True` AND `void = False`.

**Write-down documents** (`isvariance = True`) pass the isSale filter but must be excluded from revenue and profit reports.

**Document types are fully user-configurable** — names are meaningless. Always query `salesOrderDocumentTypes` dynamically.

### Revenue and Profit

- **Revenue** = `salesorder.subtotal` — line item prices × quantities. Excludes tax and balance adjustments.
- **Gross Profit** = Revenue − COGS
- **% Profit** = Gross Profit ÷ Revenue × 100 (Revenue = subtotal, not total-with-tax)
- Tax and balance adjustments (`subtotaladjustment=False`) are excluded from both revenue and profit.

### Date Fields — Critical Distinction

Reports filter by `salesorder.dateclosed` (finalization date) but display `salesorder.date` (user-editable document date). These can differ significantly. **Always filter on `dateclosed` to match built-in report behavior.**

> ⚠️ The GraphQL API `date` filter uses `salesorder.date`, not `dateclosed`. Use `dateFinalized` in GraphQL.

### Counterperson vs. Salesperson

`salesorder.salespersonuserid` (DB) = **counterperson** (misleadingly named). `salesorder.customersalespersonuserid` = commission salesperson. Using the wrong field for commission reports is a common error.

### Adjustments

- **Subtotal adjustments** (`subtotaladjustment=True`) — in `salesorder.subtotal`, revenue, and profit
- **Balance adjustments** (`subtotaladjustment=False`) — in invoice total the customer owes, NOT in subtotal or profit

Adjustments have their own date (`salesorderadjustment.date`) independent of SO dates.

### Key Concepts

- **SO Number format**: `{storeId}-{salesOrderId}` — uses `salesorderid`, not `sequentialid`
- **Invoice count**: $0 subtotal SOs excluded from count but included in dollar totals
- **Sales tax**: Always excluded from revenue and profit
- **Returns**: Negative-quantity lines. Filter by `salesorder.date` (not `dateclosed`) for return date. Dirty core returns (`DIRTY_CORE` lines) are a sub-type of return. Negative document-level adjustments are excluded from return reports by default — do not include them in return counts.
- **Customer balance vs. SO balance**: Customer's total AR differs from individual SO balance due to unallocated payments.
- **Unbalanced SOs**: Balance ≠ 0 (underpaid = positive balance, overpaid = negative). Unbalanced SO queries are **point-in-time** ("As of Date") not date-range — asking for "this month's unbalanced SOs" is the wrong framing. Unallocated payments (not linked to a specific SO via `paymentline`) reduce customer AR balance but not individual SO balances.

---

## Business Logic: Cost of Goods Sold (COGS)

> **Query reference:** Read the file `references/cogs-queries.md` for SQL patterns and the COGS by inventory type table.

### The Vehicle Cost Pool Model

1. Vehicle purchased → purchase price becomes the **cost pool** (`Vehicle.remainingCost`)
2. Parts torn from vehicle have **zero individual cost** — cost stays in the pool
3. At time of sale, a percentage of the selling price (per GL Category `percentofprice`, default 50%) is consumed from the pool as COGS — OR a `cost` field override if enabled
4. Pool reaches zero → further sales are pure profit

**For unsold parts**: `inventory.averageCost` on individual parts is zero. Use `Vehicle.remainingCost` for unsold part value.

### Line-Level Cost — Snapshot at Time of Sale

`salesorderline.averagecost` is snapshotted at time of sale — distinct from `inventory.averageCost` (current). **This is the correct field for replicating the Sales by Invoice cost column**: `SUM(salesorderline.averagecost * salesorderline.quantity)`.

### GL Scope Limitations

GL Expense entries cover inventory parts COGS but do **not** cover Job lines or Misc lines. For complete cost across all line types, use `salesorderline.averagecost` directly.

> ⚠️ `glentry.documentid` for Sales Order Line entries may reference `salesorderlineid` rather than `salesorderid`. Do not join directly on `glentry.documentid = salesorder.salesorderid`.

> ⚠️ PO Price Finalization affects COGS — adjustments not in COGS until PO is finalized.

---

## Business Logic: GL Categories

> **Schema reference:** Read the file `references/gl-categories-schema.md` for adjustment type flags table and GL query patterns.

GL categories are user-defined and control COGS rates. **Never hardcode GL category names** — always query `glcategory`. `glcategory.percentofprice` is the COGS rate. Document-level adjustments have no GL category and appear in their own section when grouping by GL category.

---

## Business Logic: Work Orders

> **Schema reference:** Read the file `references/work-order-schema.md` before writing any WO query.

Work order types are **fully user-configurable** — names are meaningless. Always check flags: `uselaborhours`, `uselaboratcost`, `workoninventory`, `pricemethod` (Markup / Cost / Customer).

### WO → SO Linkage

When a WO is finalized it generates a linked SO. Each job becomes a `Job`-type line via `salesorderline.jobid`. Report on the SO, not the WO directly. Only finalized WOs contribute to sales metrics.

### Date Filtering

`workorder.date` (document date) is used by built-in WO reports for filtering. Using `datecreated` returns a different set of WOs.

### The 3C Fields

`job.complaint` / `job.cause` / `job.correction` — standard repair documentation. Comeback detection requires a self-join on `complaint` + `customerid` + date window (no native comeback flag).

### Customer Units vs. Inventory Work

WOs with `workoninventory=False` work on customer fleet units (`workorder.customerunitid`) — separate from ITrack inventory vehicles. WOs with `workoninventory=True` work on inventory parts (Assembly Breakdown = one → many; Build Order = many → one).

---

## Business Logic: Labor Efficiency and Time Clock

> **Before writing any time clock or activity query, read `references/labor-time-clock-queries.md`.** Time clock data (`activity`, `workclock` tables) is not currently available via the GraphQL API. If the MySQL skill is available, it can provide this data. Otherwise, direct the user to built-in labor reports.

> **For efficiency calculations — check for a pre-built report via `reports` first.** Efficiency requires proportional attribution of `job.billinghours` across multiple techs on the same job. The concepts and formula are documented in `references/labor-time-clock-queries.md`.

### Store Filter — Critical Distinction

When filtering labor data by store, the correct field depends on the goal:

- **Employee's home store** (`activity.storeid`): use for attendance totals, total hours clocked, and any query where you want all employees belonging to a store — including those with no WO hours. This is the base for most labor summary queries.
- **WO's store** (`workorder.storeid` via join): use when the goal is specifically "all work performed on this store's WOs" regardless of which store the tech belongs to. Produces a different employee set — techs from other stores who worked on this store's WOs will appear; techs with no WO activity on this store won't.

**Always use `activity` as the base, not `workclock`.** The `workclock → job → workorder` join is expensive and will timeout even without a proportional subquery. Use the two-step approach documented in `references/labor-time-clock-queries.md`: query `activity` first for the employee list and clocked hours, then query `workclock` separately scoped to that userid list.

### Architecture — activity is Primary, workclock is a Subset

- **`activity`** = primary ledger — covers all employees, all time types. Columns: `activityid`, `activitytypeid`, `userid`, `clockin`, `clockout`, `wagerateid`, `storeid`, `cost`. **NOT** `starttime`/`endtime`.
- **`workclock`** = subset — WO job time only. Columns: `workclockid`, `userid`, `jobid`, `storeid`, `intime`, `outtime`, `wagerateid`, `cost`. **NOT** `clockin`/`clockout`.

**Employees without WO work have zero `workclock` records.** Always use `activity` as the base for attendance totals.

### Orgs Without Time Clock

Not all organizations use the time clock feature. If `workclock` and `activity` are empty or sparsely populated, the org likely enters labor hours manually on WO jobs rather than clocking in/out. In this case:

- `job.billinghours` is still populated — it's the manually entered billed hours figure
- Efficiency calculation is not possible (no actual clocked hours to compare against)
- Labor output can still be reported via `job.billinghours` grouped by `job.workergroupid` (assigned worker/group)
- Always check whether clock data exists before attempting an efficiency query — if `workclock` returns zero rows for the period, fall back to billing hours only and explain the limitation to the user

### The Double-Row Pattern — CRITICAL

Every clock-in writes **two rows** to `activity`:
1. `activitytypeid = 1` — attendance record (`involvement = 'Blocker'`)
2. The specific activity type (`involvement = 'Exclusive'` or `'Open'`)

**Filter to `activitytypeid = 1` for attendance totals** — summing all rows doubles every hour.

### activitytype.workordertypeid

Non-null = WO-linked type. Null = general activity. Query `activitytype` to understand the mapping before building category breakdowns.

### Billing Hours vs. Clocked Hours

- `job.billinghours` = hours billed (flag hours)
- `job.expectedhours` = flat-rate guide estimate (may be 0)
- When "Bill Actual Hours" is on, `billinghours` = clocked — efficiency ratio is meaningless. Use `expectedhours / clocked_hours` instead.
- `workclock.cost` = internal labor cost. `job.laborcharge` = what customer was billed. Do not confuse them.

### Shared Job Attribution

Multiple techs on one job: billing hours attributed proportionally `billinghours × (tech_clock / total_clock)`. Built-in reports apply this; raw queries do not.

---

## Business Logic: Purchase Orders

> **Schema reference:** Read the file `references/purchasing-schema.md` for PO field tables, posting fields, and SQL patterns.

Two-level structure: **PO** and **Postings** (receiving events). Lifecycle: Active → Done Receiving → Prices Finalized. COGS accuracy requires price finalization.

**Key date distinction**: `purchaseorderhistory.postdate` for received items; `purchaseorder.date` for PO-level queries.

**Vendor code vs. manufacturer code**: Separate entities. Use `vendor.code` for vendor filtering.

---

## Business Logic: Accounts Receivable

Payments in `payment` table; allocation to SOs in `paymentline`. Unapplied payments reduce customer AR balance but not individual SO balances.

**AR Aging** is point-in-time ("As of Date"). `term.chargedays` can be null for calendar-based terms — due date calculation not always possible.

---

## Business Logic: Picking and Shipping

> **Schema reference:** Read the file `references/sales-order-schema.md` for shipping method and pick status field tables.

"Deliveries", "shipping", "picking" = **outgoing sales shipments**, not incoming PO receipts. Clarify if unclear.

Shipping methods are user-configured with behavioral flags — `autoPick`, `deliverWhenPicked` (Will Call), `inHouse`, `pickup`. Always query `shippingMethods`.

Pick progress: `quantityPicked`, `quantityReady`, `quantityDelivered`, `quantityMissing` on SalesOrderLine. `totalQuantityReady` / `totalQuantityMissing` on SalesOrder as rollups.

The Deliveries screen shows all document types (SOs, WOs, TOs, POs) — confirm which types the user wants.

Run sheets are not in the GraphQL API. If the MySQL skill is available, see `shipping-delivery.md` for run sheet queries. Otherwise, direct the user to the Deliveries screen in ITrack Desktop.

---

## Business Logic: Inventory Structure

> **Schema reference:** Read the file `references/inventory-schema.md` for the full inventory field table, status codes, stock category, and inventory type flags.

> **Before calculating inventory age, call `get_current_date` first.**

**Status** — live inventory = `A` or `H`. Filter to `status IN ('A','H')` for current stock.

**Stock Category** — `STANDARD` (teardown parts), `REPLENISHABLE` (restockable, averageCost updates), `MISC` (one-off).

**Inventory Types** — user-configurable, names meaningless. Check `inventorytype.vehicleunit` to distinguish vehicles from parts.

**`inventory.averagecost`** — valuation amount. Resets to zero at zero qty by default. Not the same as `salesorderline.averagecost` (snapshotted at sale).

**`inventory.cost`** — cost override, NOT the valuation amount.

### Custom Fields — Two Systems

**Type-specific fields** (`typedata1`–`4`) — up to 4 per type. **Q&A fields** — stored in `optionsValues`, richer (choice lists, date/currency/number types, public flag). See **`itrack-enterprise-custom-fields`** skill for querying Q&A data.

### Manufacturer vs. Parent Manufacturer

Assembly/Parent = what the part fits or came from. True/Item = who made it. Which matters depends on part type — mechanical assemblies care about item manufacturer; body parts care about source vehicle year/make/model. For universal or generic parts, neither may be meaningful. Confirm with the organization or part type context which association is relevant before building a query. Parts linked via `inventory.vehicleid` expose `vehicleMake`, `vehicleModel`, `vehicleYear`, `vehicleVin` as API convenience fields.

### HTP Avg

Available in the app but not queryable through the current MCP connector. Direct users to the Part Management screen.

---

## Business Logic: Vehicles

> **Schema reference:** Read the file `references/inventory-schema.md` for vehicle field table, cost pool queries, and core fields.

Vehicles are inventory records with `inventorytype.vehicleunit = True` plus a `vehicle` table record. Heavy truck yards say "vehicles"; heavy equipment yards say "units." `vehicle.mileage` tracks hours for equipment organizations.

### "A Truck" — Disambiguate Before Querying

When a user asks about "a truck" or "a unit," the context determines which data model applies:

- **"What's going on with truck #42" / "show me the status of our 2019 Freightliner"** — likely referring to an **inventory vehicle** (a stock unit or teardown vehicle in ITrack). Query `inventory` / `vehicle` records.
- **"The truck in bay 3" / "work being done on a customer's truck" / "service on a customer unit"** — likely referring to a **customer unit on a work order** (`workorder.customerunitid`). Customer units are the customer's own fleet, not ITrack inventory.
- **"Parts coming off that truck"** — inventory vehicle in teardown context. Query parts via `inventory.vehicleid`.

If the context is ambiguous, ask: *"Are you asking about a vehicle in your inventory, or a customer's vehicle that's in for service?"*

### Clarify Vehicle Type Before Querying

- **Whole units for sale** — sold intact. No parts to pull. Value = asking price.
- **Parts/teardown vehicles** — cost pool model applies.

Check `inventorytype` configuration. Query `inventoryTypes` for current installation.

### Parts Still on a Vehicle ("Unpulled" Parts)

**Not yet inventoried**: value = `Vehicle.remainingCost`. No individual part records exist.

**Partially dismantled**: `Vehicle.remainingCost` = unallocated cost. Unsold parts have zero `averageCost` — cost moves from pool to part only at time of sale.

**Inventoried but in holding location**: parts in system at yard/dock location. Names vary by org.

**Do not use `inventory.averageCost` on parts to assess value of unsold teardown parts.**

Non-A/H vehicles with remaining cost pool skew inventory value totals — flag separately.

---

## Business Logic: Inventory Locations, Counts and Moves

> **Schema reference:** Read the file `references/locations-schema.md` for location tables, flag patterns, and transfer order fields.

> **Trigger note:** Warehouse management, bin locations, inventory counts, cycle counts, inventory moves, barcode scanning all map here.

### Stores and the SKU Model

Each `inventory` record belongs to one store. Clone = same `inventoryId` at multiple stores (shared global fields, per-store qty/price). Replicate = new independent `inventoryId`. When aggregating across stores, decide whether to group by `inventoryId` (clones combined) or `inventory.id` (per-store records).

### Transfer Orders

Formal mechanism for moving inventory between stores. Lifecycle: `NEW → TRANSMITTED → ACKNOWLEDGED → FILLED → SHIPPED`. Transfer activity does NOT count as a "sale" for unmoved inventory analysis.

### Location Flags

`countable` (in count sheets), `sellable` (can be picked), `allowinventory` (can store items). Filter to `allowinventory=True` for valid storage locations. Organizational container rows typically have all three False.

**Variance location** may be renamed by organizations — identify by flags (`countable=False`, `sellable=False`), not by name. Items routed there due to quantity discrepancies must be resolved: move to correct bin, write off via Inventory Adjustment, or return to correct location.

### Inventory Aging and Lifecycle

Age = days from `purchaseorderhistory.postdate` to today. Only items with a PO receive date have calculable age.

**Unmoved inventory** = not on any SO or WO job in a period. Distinct from aging — unmoved measures absence of sales activity.

Use age + unmoved + sales history + profitability together for scrapping vs. repricing decisions.

---

## Business Logic: Cores

> **Schema reference:** Read the file `references/inventory-schema.md` for core field tables and vendor core status enum.

Core exchange = deposit/return model: customer pays a core charge at purchase, returns old part, gets deposit back.

**Inherent Core** (`INHERENT_CORE`) — deposit charge on SO line at time of sale. Not revenue — it's a deposit. Not a physical inventory item.

**Dirty Core** (`DIRTY_CORE`) — returned worn part. Negative SO line (credit). `salesorderline.corestatus` tracks vendor return: `NA`, `AVAILABLE`, `PROCESSED`, `REJECTED`.

**Vendor Cores** — yard's obligation to return cores to vendor. Tracked via `coreRequiredToVendor`, `coreClass`, `daysToReturnCoreToVendor`.

Core charges are excluded from standard profit metrics. In part-level aggregations, inherent cores roll into the parent item's total.

