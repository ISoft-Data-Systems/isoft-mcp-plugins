# Purchasing Schema Reference

## PO Lifecycle

| Stage | GraphQL field | Meaning |
|---|---|---|
| Active | `PurchaseOrder.doneReceiving = false` | Open; items can still be received |
| Done Receiving | `PurchaseOrder.doneReceiving = true` | Receiving closed |
| Prices Finalized | `PurchaseOrder.pricesFinalized = true` | Pricing locked; COGS final |

## Key PO Fields

| GraphQL field | Notes |
|---|---|
| `PurchaseOrder.vendor` | Source vendor |
| `PurchaseOrder.store` | Destination store |
| `PurchaseOrder.customer` | Drop-ship to customer |
| `PurchaseOrder.doneReceiving` | |
| `PurchaseOrder.pricesFinalized` | |
| `PurchaseOrder.void` | |
| `PurchaseOrder.postings` | Per-receiving-event records |

## PO Line Fields

| GraphQL field | Meaning |
|---|---|
| `PurchaseOrderLine.quantity` | Total ordered |
| `PurchaseOrderLine.quantityReceived` | Total received across all postings |
| `PurchaseOrderLine.price` | Unit price |
| `PurchaseOrderLine.inventory` | Linked inventory record |
| `PurchaseOrderLine.total` | price × quantity |
| Qty received per posting | *(not in GraphQL API)* — MySQL skill only; see `purchasing.md` |
| Cost per unit per posting | *(not in GraphQL API)* — MySQL skill only; see `purchasing.md` |

## Posting Fields (PurchaseOrderPosting)

| GraphQL field | Meaning |
|---|---|
| `PurchaseOrderPosting.number` | Sequence number within the PO |
| `PurchaseOrderPosting.postDate` | Date of receiving event — used as date filter for received items |
| `PurchaseOrderPosting.approved` | Whether an admin has approved/locked this posting |
| `PurchaseOrderPosting.vendorInvoiceNumber` | Vendor's invoice reference |
| `PurchaseOrderPosting.vendorInvoiceDate` | Vendor's invoice date |
| `PurchaseOrderPosting.vendorPackingListNumber` | Vendor packing list reference |
| `PurchaseOrderPosting.paymentDueDate` | Date payment is due for this vendor invoice |
| `PurchaseOrderPosting.discountDate` | Last date vendor will give a discount |
| `PurchaseOrderPosting.term` | Payment terms for this posting |
| `PurchaseOrderPosting.receivingLocation` | Default location to receive inventory into |
| `PurchaseOrderPosting.postedByUser` | Who performed the receiving |
| `PurchaseOrderPosting.store` | Store receiving inventory — may differ from PO store |

## Posting Line Fields (PurchaseOrderPostingLine)

| GraphQL field | Meaning |
|---|---|
| `PurchaseOrderPostingLine.received` | Quantity received in this posting |
| `PurchaseOrderPostingLine.cost` | Cost per unit in this posting |
| `PurchaseOrderPostingLine.pay` | AP flag — whether to pay for this line |
| `PurchaseOrderPostingLine.inventoryDebitGlAccount` | GL account debited for inventory |
| `PurchaseOrderPostingLine.inventoryCreditGlAccount` | GL account credited for inventory |

## PO vs SO Structure Differences

| Concept | Sales Order | Purchase Order |
|---|---|---|
| Document-level adjustments | Supported (subtotal or balance) | Supported (always add to cost) |
| Line-level adjustments | Not supported | Supported per posting line |
| Multiple receiving events | Not applicable | Multiple postings per PO |
| Approval workflow | Document-level finalization | Per-posting approval + PO price finalization |
| Destination | Always to a customer | To a store, or direct to customer (drop-ship) |

## Outstanding Unreceived Lines

Lines where `PurchaseOrderLine.quantityReceived < PurchaseOrderLine.quantity` on POs where `PurchaseOrder.doneReceiving = false` and `PurchaseOrder.void = false`. Use these GraphQL fields to identify and surface outstanding quantity:

```graphql
{
  purchaseOrders(filter: { doneReceiving: false, void: false }) {
    nodes {
      purchaseOrderId
      store { name }
      lines {
        nodes {
          quantity
          quantityReceived
          inventory { tagNumber description }
        }
      }
    }
  }
}
```

Filter client-side (or via filter if available) to lines where `quantityReceived < quantity` to get outstanding lines.
