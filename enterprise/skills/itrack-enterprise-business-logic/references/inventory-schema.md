# Inventory Schema Reference

## Inventory Status Codes

| Status | Meaning |
|---|---|
| `A` | Available — in stock and sellable |
| `B` | Bidding — vehicle being bid on |
| `C` | Crushed — vehicle scrapped |
| `D` | Deleted — soft-deleted |
| `H` | Hold — on hold, not available for sale |
| `S` | Sold |
| `T` | Transferred |

**Live inventory** = status `A` or `H`

## Stock Category

| Value | Meaning |
|---|---|
| `STANDARD` | From a vehicle teardown. Marked Sold when quantity hits zero. |
| `REPLENISHABLE` | Expected to be restocked. NOT marked Sold on depletion. Average cost updates with purchases. |
| `MISC` | Not from a vehicle, not replenishable. |

## Inventory Type Flags

Always query `inventoryTypes` — names and IDs are user-configurable.

| GraphQL field | Meaning |
|---|---|
| `InventoryType.vehicleUnit` | True = whole vehicle; False = part/component |
| `InventoryType.pickable` | Can be picked for delivery |
| `InventoryType.active` | Currently in use |

## Key Inventory Fields

| GraphQL field | Notes |
|---|---|
| `Inventory.tagNumber` | User-visible tag. Not unique across stores. |
| `Inventory.inventoryId` | SKU — may be shared across stores (clone) |
| `Inventory.id` | Unique per store — the primary key |
| `Inventory.storeId` | Which store this record belongs to |
| `Inventory.quantity` | On-hand total across all locations |
| `Inventory.quantityOnHold` | Held by open demand documents |
| `Inventory.quantityOnOrder` | On open POs |
| `Inventory.averageCost` | Valuation amount — resets to 0 at zero qty by default |
| `Inventory.cost` | Cost override — NOT valuation; used in percent-of-price override |
| `Inventory.averageCoreCost` | Average core cost |
| `Inventory.retailPrice` | Retail tier |
| `Inventory.wholesalePrice` | Wholesale tier |
| `Inventory.jobberPrice` | Jobber tier |
| `Inventory.listPrice` | Manufacturer list price |
| `Inventory.defaultVendor` | Default vendor. Null for auction vehicles and misc items. |
| `Inventory.vehicle` | Source vehicle |
| `Inventory.vehicleMake` | Source vehicle make (convenience field) |
| `Inventory.vehicleModel` | Source vehicle model (convenience field) |
| `Inventory.vehicleYear` | Source vehicle year (convenience field) |
| `Inventory.vehicleVin` | Source vehicle VIN (convenience field) |
| `Inventory.glCategory` | GL category controlling COGS rate |
| `Inventory.stockCategory` | STANDARD / REPLENISHABLE / MISC |
| `Inventory.status` | StatusEnum |
| `Inventory.userStatus` | User-defined single character |
| `Inventory.dateEntered` | Record creation date |
| `Inventory.typeField1`–`4` | Type-specific custom fields (More Labels) |
| `Inventory.parentManufacturer` | Assembly/application manufacturer |
| `Inventory.parentModel` | Assembly/application model |
| `Inventory.manufacturer` | True manufacturer |
| `Inventory.model` | True model |

## Active Inventory with No Location

Inventory records with status `A` or `H`, quantity > 0, and no `InventoryLocation` assignment represent items that exist in stock but have no bin location recorded — a potential warehouse management gap. Location data is not currently available via the GraphQL API. If the MySQL skill is available, see `warehouse.md` for location queries.

## Vehicle Fields

| GraphQL field | Meaning |
|---|---|
| `Vehicle.stockNumber` | User-visible stock number |
| `Vehicle.purchaseDate` | When purchased |
| `Vehicle.dismantleDate` | When dismantling began (can be null even if dismantled=True) |
| `Vehicle.dismantled` | Whether torn down |
| `Vehicle.dismantler` | Who performed teardown |
| `Vehicle.receivedDate` | When physically received |
| `Vehicle.glCategory` | GL category for the vehicle |
| `Vehicle.inventoryGlCategory` | GL category for parts torn from this vehicle |
| `Vehicle.remainingCost` | Unallocated cost pool balance — use this for unsold parts value |
| `Vehicle.consumedCost` | Cost allocated to sold parts |
| `Vehicle.allocatedCost` | Total cost allocated so far |

> **For unsold parts value:** Use `Vehicle.remainingCost`, NOT `inventory.averageCost` on individual parts. Part-level average cost is zero for unsold teardown parts — cost only moves from the vehicle pool to a part at time of sale.

## Core-Related Fields on Inventory

| GraphQL field | Meaning |
|---|---|
| `Inventory.isCore` | Whether classified as a core item |
| `Inventory.coreClass` | Groups interchangeable cores for vendor exchange |
| `Inventory.retailCorePrice` | Core charge at retail tier |
| `Inventory.jobberCorePrice` | Core charge at jobber tier |
| `Inventory.wholesaleCorePrice` | Core charge at wholesale tier |
| `Inventory.distributorCorePrice` | Core charge at distributor tier |
| `Inventory.averageCoreCost` | Average cost of the core |
| `Inventory.coreCost` | Core replacement cost |
| `Inventory.coreRequired` | Customer core return required |
| `Inventory.coreRequiredToVendor` | Vendor core return required |
| `Inventory.daysToReturnCore` | Grace period for customer return |
| `Inventory.daysToReturnCoreToVendor` | Grace period for vendor return |

## Vendor Core Status (on SalesOrderLine)

| CoreStatusEnum | Meaning |
|---|---|
| `NA` | Not applicable |
| `AVAILABLE` | Dirty core received, available to return to vendor |
| `PROCESSED` | Returned to vendor |
| `REJECTED` | Rejected by vendor |
