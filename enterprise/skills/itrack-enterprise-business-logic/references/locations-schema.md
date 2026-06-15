# Locations Schema Reference

## Location Table

| GraphQL field | Meaning |
|---|---|
| `Location.code` | Short location code |
| `Location.description` | Full location name |
| `Location.parentLocation` | Parent — enables nested hierarchies |
| `Location.countable` | Included in inventory count sheets |
| `Location.sellable` | Can be picked and sold |
| `Location.allowInventory` | Inventory can be stored here |

## InventoryLocation Table

| GraphQL field | Meaning |
|---|---|
| `InventoryLocation.quantity` | Qty at this location |
| `InventoryLocation.holdQuantity` | Qty on hold |
| `InventoryLocation.permanent` | Home location flag |
| `InventoryLocation.rank` | Display order |

## Quantity Calculations

```
Total QOH              = Inventory.quantity
At Location QOH        = InventoryLocation.quantity
At Location Available  = InventoryLocation.quantity - InventoryLocation.holdQuantity
```

## Typical Location Flag Patterns

| Location type | countable | sellable | allowinventory | Notes |
|---|---|---|---|---|
| Warehouse/bin | ✓ | ✓ | ✓ | Standard storage |
| Dock / Receiving | ✗ | ✓ | ✓ | Sellable but not counted |
| Shipping | ✓ | ✓ | ✓ | Staged for outbound |
| Shop / Work area | ✓ | ✗ | ✓ | In use — not available for sale |
| Variance | ✗ | ✗ | ✓ | Discrepancy holding area |
| Yard | ✗ | ✓ | ✓ | Outdoor storage |
| Organizational row | ✗ | ✗ | ✗ | Container only — no actual storage |

> Variance location may be renamed by organizations (e.g. "Mistake!!!", "Discrepancy"). Identify by flags, not name.

> Organizational container locations (parent rows A-K etc.) typically have all three flags False. Filter to `allowinventory=True` for valid storage locations.

## Transfer Order Fields

| GraphQL field | Meaning |
|---|---|
| `TransferOrder.sourceStore` | Store sending inventory |
| `TransferOrder.destinationStore` | Store receiving inventory |
| `TransferOrder.ordered` | Order placed |
| `TransferOrder.doneReceiving` | Destination has received |
| `TransferOrder.orderStatus` | NEW → TRANSMITTED → ACKNOWLEDGED → FILLED → SHIPPED |
| `TransferOrder.void` | |
| `TransferOrder.sourceComment` | Notes from sending store |
| `TransferOrder.destinationComment` | Notes from receiving store |

## SKU Model — Multi-Store Identity

| Concept | GraphQL field | Meaning |
|---|---|---|
| Store-specific record | `Inventory.id` | Unique per store — primary key |
| SKU / item identity | `Inventory.inventoryId` | Universal — may be shared across stores (clone) |
| SKU record | `Inventory.skuType` | `inventories` list = all store instances of this SKU |
| Store | `Inventory.storeId` | Which store this record belongs to |

**Clone** = same `inventoryId` at multiple stores. **Replicate** = new independent `inventoryId`. When aggregating across stores, decide whether to group by `inventoryId` (clones combined) or `inventory.id` (per-store records).
