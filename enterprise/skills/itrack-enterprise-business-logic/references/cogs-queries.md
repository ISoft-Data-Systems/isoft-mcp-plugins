# COGS Query Reference

## COGS by Inventory Type

| Type | How COGS is determined |
|---|---|
| Standard parts (vehicle teardown) | Percent of selling price per GL Category (`glcategory.percentofprice`) OR cost override if set |
| Complete vehicle (`inventorytype.vehicleunit = True`) | 100% of vehicle cost pool consumed at sale |
| Miscellaneous parts (no vehicle) | Average cost at time of sale |
| Replenishable parts | Average cost, updated with each purchase |
| Cores (`INHERENT_CORE` / `DIRTY_CORE`) | Excluded from standard profit metrics |

---

## Line-Level COGS (for replicating Sales by Invoice report)

`SalesOrderLine.averageCost` is the cost snapshotted at time of sale. This is what the built-in Sales by Invoice report uses and produces complete results across all line types.

```graphql
salesOrders(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: { storeId: [1], isSale: true, date: { gte: "2026-01-01", lte: "2026-01-31" } }
) {
  items {
    salesOrderId
    lines(pagination: { pageNumber: 1, pageSize: 200 }) {
      items {
        type
        quantity
        averageCost
      }
    }
  }
}
```

Compute `SUM(averageCost * quantity)` per SO client-side. Exclude `INHERENT_CORE` and `DIRTY_CORE` line types from profit metrics.

---

## GL-Based COGS (inventory parts only)

> ⚠️ GL Expense entries only cover inventory lines. Job lines (labor) and Misc lines are not covered. Use the line-level approach above for complete coverage.

> ⚠️ `glentry.documentid` for `Sales Order Line` entries = `innodb_salesorderid` (the SO's true primary key), not the user-facing `salesOrderId`. Join to `salesorder.innodb_salesorderid`.

GL entry data is not currently available via the GraphQL API. If the MySQL skill is available, see `accounting.md` for GL-based COGS query patterns.

---

## Vehicle Cost Pool — Active Vehicle Value

`Vehicle.remainingCost` is available via GraphQL — use it for active vehicle cost pool value:

```graphql
vehicles(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: { stores: [1], statuses: ["A", "H"] }
) {
  items {
    id
    stockNumber
    year
    make
    model { name }
    remainingCost
    consumedCost
    allocatedCost
  }
}
```

Filter client-side to `remainingCost > 0` to identify vehicles with unallocated cost pool balance.

---

## Vehicle Cost Pool — Stranded Cost Warning

Vehicles with status other than `A` or `H` but with remaining cost represent a data quality issue — cost pool not fully consumed before the vehicle was closed out.

> Not available via GraphQL API — vehicle status and cost pool cannot be filtered in combination server-side for non-active statuses. If the MySQL skill is available, see `accounting.md` — "Vehicle Cost Pool Stranded Cost" query.

If this comes up, direct the user to run an inventory audit or contact support to review closed vehicles with residual cost.
