# GL Categories Schema Reference

## Adjustment Type Flags

| GraphQL field | Meaning |
|---|---|
| `AdjustmentType.scope` | `Line` = individual line items only; `Document` = document level only; `Both` = either level |
| `AdjustmentType.subtotalAdjustment` | `True` = Subtotal Adjustment (in subtotal/revenue); `False` = Balance Adjustment (in total owed, not revenue) |
| `AdjustmentType.showOnSalesOrders` | Available on SOs |
| `AdjustmentType.showOnPurchaseOrders` | Available on POs |
| `AdjustmentType.taxable` | Subject to sales tax |
| `AdjustmentType.atCost` | Recorded at cost (write-down/variance contexts) |
| `includeinsales`, `printable` | *(not in GraphQL API)* — MySQL skill only |

---

## GL Contexts

GL categories can have different accounting rules per document context. The same GL category may post to different accounts depending on whether the transaction is a sale, purchase, or adjustment. Always introspect the `GlCategory` type for available context fields.

---

## GL Entry Queries

GL entry data is not currently available via the GraphQL API. If the MySQL skill is available, see `accounting.md` for verified query patterns.

**Key concepts regardless of query method:**

- GL entries are linked to source documents via `documentType` + `documentId` + `documentLineId`
- `effectiveDate` on the transaction record controls which reporting period an entry falls in — not `createdDate`
- Every transaction is double-entry — matching debits and credits. Filter by transaction type when summing

**GL Account Types:**

| Type | Meaning |
|---|---|
| `Revenue` | Sales revenue accounts |
| `Expense` | COGS and expense accounts |
| `Asset` | Inventory and asset accounts |
| `Liability` | Accounts payable and liability |

---

## Document Linkage

When working with GL entries, `documentType` and `documentId` identify the source document. Key notes:

- For `Sales Order Line` entries, `documentId` = `innodb_salesorderid` (the SO's true primary key), not the user-facing `salesOrderId`.
- Filtering by a specific SO may also surface related WO and payment GL entries

> ⚠️ **GL entries post at WO finalization, not at clock-out**: `Work Order Job` and `Work Order Labor` GL entries both post when the WO is finalized — `effectiveDate` = finalization date, not the date the tech clocked in. Filtering GL by `effectiveDate` will not match clock-based date ranges for open WOs. Use clock times for attendance/labor filtering; use GL for amounts only.

---

## GL Document Types — Full Reference

| Document type | Document ID refers to | Line ID refers to | What it records |
|---|---|---|---|
| `Sales Order` | SO ID | null | Tax and AR at SO level |
| `Sales Order Line` | `innodb_salesorderid` (SO true PK) | SO line ID | Parts revenue and COGS per SO line |
| `Sales Order Adjustment` | SO ID | Adjustment line ID | Discounts, credits, adjustments |
| `Work Order Job` | WO ID | Job ID | Labor revenue and shop fee per job — posts at WO finalization |
| `Work Order Labor` | WO ID | Clock entry ID | Per-tech labor cost — posts at WO finalization, not at clock-out |
| `Work Order Part` | WO ID | WO part line ID | Parts revenue and COGS per part pulled to WO |
| `Purchase Order Line` | PO ID | PO line ID | Purchasing entries |
| `Payment` | Payment ID | null | Customer payment entries |

> ⚠️ `Work Order Labor` line ID = clock entry ID — enables tracing GL cost entries back to individual technician clock entries for point-in-time cost attribution.
