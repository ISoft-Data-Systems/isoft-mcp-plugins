# Sales Order Schema Reference

## GraphQL Query Gotchas

> ⚠️ **`SalesOrderFilter.ids` takes `SalesOrderKey` objects, not scalars**: `ids: [43685]` (int) and `ids: ["43685"]` (string) both fail schema validation. Correct form: `ids: [{ storeId: 2, salesOrderId: 43685 }]`. Always provide both `storeId` and `salesOrderId`.

> ⚠️ **`SalesOrderFilter.date` uses comparison operators, not `from/to`**: `date: { from: "...", to: "..." }` is invalid. Use `date: { gte: "2026-04-10", lte: "2026-04-10" }` for date range filtering.

> ⚠️ **`SalesOrderFilter.customerFilter` throws a DB error at runtime**: `customerFilter: { ids: [19497] }` passes schema validation but returns `ER_BAD_FIELD_ERROR`. Workaround: filter by customer in application logic after fetching, or use `SalesOrderKey` if SO IDs are known.

> ⚠️ **`SalesOrder.lines` is a paginated list, not a bare array**: Querying `lines { id type ... }` directly fails. Use: `lines(pagination: { pageNumber: 1, pageSize: 100 }) { totalItems items { ... } }`.

> ⚠️ **`SalesOrderFilter.customerFilter.ids` expects a list even for a single customer**: There is no singular `id` field — always use `ids: [value]`.

> ⚠️ **`SalesOrderFilter` has no `finalized`, `void`, or `isSale` fields**: These are not valid filter fields and will fail schema validation. Filtering on closed or void status is not currently available via the GraphQL API. If the MySQL skill is available, filter directly on the `salesorder` table.

## Sales Order Data Structure

| Field | GraphQL field | Notes |
|---|---|---|
| SO number | `SalesOrder.salesOrderId` | Displayed as `{storeId}-{salesOrderId}` |
| Document type | `SalesOrder.documentType` | Controls accounting behavior via flags — never rely on name |
| Document date | `SalesOrder.date` | User-editable |
| Finalization date | `SalesOrder.dateFinalized` | System-set — used for date range filtering in reports |
| Counterperson | `SalesOrder.counterperson` | Who processed the order |
| Salesperson | `SalesOrder.salesperson` | Commission salesperson; defaults from customer |
| Tax | `SalesOrder.tax` | Never included in revenue or profit |
| Subtotal | `SalesOrder.subtotal` | Revenue basis |
| Total | `SalesOrder.total` | subtotal + tax + balance adjustments |
| Paid | `SalesOrder.appliedPaymentTotal` | Amount collected |
| Balance | `SalesOrder.balance` | Total − Paid |
| Void | `SalesOrder.void` | Excluded from all metrics |
| Finalized | `SalesOrder.finalized` | Must be True for a sale to count |
| Is sale | `SalesOrder.isSale` | `affectsInventory AND finalized AND NOT void` |
| Payment terms | `SalesOrder.term` | |
| Customer PO# | `SalesOrder.purchaseOrderNumber` | Customer's own PO reference |
| Ship via | `SalesOrder.shippingMethod` | Shipping method |
| Tracking# | `SalesOrder.shippingTrackingNumber` | Carrier tracking |
| Allow partial ship | `SalesOrder.allowPartialShipment` | |
| Allow backorder | `SalesOrder.allowBackorder` | |
| Qty ready | `SalesOrder.totalQuantityReady` | Picked and staged |
| Qty missing | `SalesOrder.totalQuantityMissing` | Not yet picked |

## Sales Order Document Type Flags

Always query dynamically: `{ salesOrderDocumentTypes { id name affectsInventory isLostSale isWriteDown holdsQty } }`

| GraphQL field | Meaning |
|---|---|
| `affectsInventory` | Triggers accounting and inventory at finalization |
| `isLostSale` | Tracks missed sales — no inventory or revenue impact |
| `isWriteDown` | Write-down document — always at cost, never sale price |
| `holdsQty` | Reserves quantity without completing a sale — does NOT mean the document is on hold |

## Sales Order Line Fields

| GraphQL field | Notes |
|---|---|
| `SalesOrderLine.price` | What was charged — source of truth for revenue |
| `SalesOrderLine.retailPrice` | Snapshot of retail at time of sale |
| `SalesOrderLine.wholesalePrice` | Snapshot of wholesale |
| `SalesOrderLine.jobberPrice` | Snapshot of jobber |
| `SalesOrderLine.distributorPrice` | Snapshot of distributor |
| `SalesOrderLine.customerPrice` | Customer price level — used for deviation analysis |
| `SalesOrderLine.averageCost` | Cost snapshotted at time of sale — distinct from `Inventory.averageCost` |
| `SalesOrderLine.type` | INVENTORY, JOB, INHERENT_CORE, DIRTY_CORE, MISC |
| `SalesOrderLine.coreStatus` | Vendor core return status: NA, AVAILABLE, PROCESSED, REJECTED |
| `SalesOrderLine.quantity` | Negative = return |
| `SalesOrderLine.total` | price × quantity |
| `SalesOrderLine.job` | Links to WO job (WO-sourced lines) |
| `SalesOrderLine.inventory` | Links to inventory record |
| `SalesOrderLine.enteredBy` | Who entered this line |
| `SalesOrderLine.deliveryStatus` | *(not in GraphQL API)* |
| `SalesOrderLine.quantityPicked` | Pulled from shelf |
| `SalesOrderLine.quantityReady` | Picked and staged |
| `SalesOrderLine.quantityDelivered` | Confirmed delivered |
| `SalesOrderLine.quantityMissing` | Not yet located |
| `SalesOrderLine.deliveryDate` | Delivery date |
| `SalesOrderLine.creditSalesOrderLine` | Links return line to original sale |

## Customer Fields (commonly used on SalesOrder)

> ⚠️ `Customer` has no `name` field. Use `companyName` for the business name and `contactName` for the individual contact. Using `customer { name }` will fail schema validation.

> ⚠️ Customer address fields are NOT flat on `Customer` — they are nested under `billingAddress` which is an `Address` type. Use `billingAddress { address1 address2 city state zip country }`.

| GraphQL field | Meaning |
|---|---|
| `Customer.companyName` | Business/company name |
| `Customer.contactName` | Individual contact name |
| `Customer.id` | Customer ID |
| `Customer.phoneNumber` | Primary phone |
| `Customer.email` | Email address |
| `Customer.balance` | Current AR balance |
| `Customer.billingAddress { address1 address2 city state zip }` | Billing address — nested `Address` type |
| `Customer.addresses` | All addresses including shipping — list of `CustomerAddress` |

**CustomerFilter fields:** `ids`, `active`, `balance`, `companyName`, `contactName`, `email`, `phoneNumber`, `keyword`

```graphql
{
  customers(pagination: { pageNumber: 1, pageSize: 10 }, filter: { keyword: "BADGER" }) {
    items {
      id companyName
      billingAddress { address1 city state zip }
    }
  }
}
```

**Balance formula:**
- `SalesOrder.total` = subtotal + tax + balance adjustments
- `SalesOrder.appliedPaymentTotal` = amount collected
- `SalesOrder.balance` = total − paid
- **Subtotal adjustments** — in subtotal and revenue
- **Balance adjustments** — in total owed, NOT in subtotal or revenue

## Payment Terms Fields

| GraphQL field | Meaning |
|---|---|
| `Term.name` | Display name (e.g. "Net 30", "COD") |
| `Term.code` | Short code |
| `Term.chargeDays` | Days until payment due — **nullable** for calendar-based terms |
| `Term.chargeDayOfMonth` | Day of month due (for calendar-based terms) — nullable |
| `Term.chargePercent` | Finance charge rate |
| `Term.chargeFromPurchase` | Whether charge period starts from purchase date |
| `Term.discountDays` | Days within which discount applies — nullable |
| `Term.discountDayOfMonth` | Day of month for discount cutoff — nullable |
| `Term.discountPercent` | Discount rate |

When `chargeDays` is null, use `chargeDayOfMonth` for due date calculation.

## Counterperson vs Salesperson

| Concept | GraphQL field |
|---|---|
| Counterperson | `SalesOrder.counterperson` |
| Salesperson (commission) | `SalesOrder.salesperson` |
| Line entry person | `SalesOrderLine.enteredBy` |

## Shipping Method Flags

Always query: `{ shippingMethods { ... } }`

| GraphQL field | Meaning |
|---|---|
| `ShippingMethod.name` | User-defined name |
| `ShippingMethod.autoPick` | Auto-marks items as picked |
| `ShippingMethod.deliverWhenPicked` | Marks delivered immediately on pick (e.g. Will Call) |
| `ShippingMethod.inHouse` | Company-owned delivery |
| `ShippingMethod.pickup` | Customer picks up at store |
| `ShippingMethod.showOnPicking` | Shows in Picking screen |
| `ShippingMethod.showOnRunSheet` | Shows on run sheets |
| `ShippingMethod.showOnDocument` | Shows in shipping method dropdown |
| `ShippingMethod.trackingUrl` | Carrier tracking URL template |
