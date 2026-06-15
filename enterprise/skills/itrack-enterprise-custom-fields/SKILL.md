---
name: itrack-enterprise-custom-fields
description: >
  Optimized querying of Custom Fields (Q&A fields) stored on customer, vendor, inventory,
  and vehicle records in ITrack Enterprise, using the Enterprise-Api-Extension MCP.
  Use this skill whenever a user asks for customer, vendor, inventory part, or vehicle
  information that doesn't appear on the main record — it's likely stored as either a
  Custom Field (Q&A field) or a type-specific Additional Field (typeField1–4). Both must
  be checked before concluding a field doesn't exist.
  Never mention "optionsValues" or "optionsvalues table" to users; always say "Custom Fields".
  When this skill requires paginating across multiple customer or vendor records (e.g. scanning
  for a Custom Field value), also use the enterprise-query skill for correct query and pagination patterns.
metadata:
  author: ISoft Data Systems, Inc.
  version: 0.2.3
  mcp-server: Enterprise-Api-Extension
  category: universal
---

# Custom Fields (Q&A Fields) in ITrack Enterprise

---

## Overview

Custom Fields (also called Q&A fields) are user-defined fields that extend customer, vendor, inventory, and vehicle records. Each database has its own unique set of Custom Fields — users will refer to them by their label names (e.g., "Fleet Size", "Territory", "Payment Terms").

**Terminology:** The software now uses the term "Custom Fields" as the preferred name for these fields. Older users may refer to them as "Q&A fields" or "flex fields" — these are the same feature under legacy names. Always respond using "Custom Fields" regardless of which term the user uses, but recognize all three as valid references to the same thing.

**Key rule:** If a user asks for data that isn't a standard field (name, address, phone, etc.), assume it's either a Custom Field or a type-specific Additional Field (`typeField1`–`4`) and query accordingly. Check both systems before concluding the field doesn't exist.

Never refer to the underlying table by its database name. Always say "Custom Fields" or "Q&A fields."

---

## Two Systems for Extended Inventory Fields

Inventory parts have **two separate systems** for storing non-standard data. When a user asks for a field that isn't on the standard inventory record, check both before concluding it doesn't exist:

### 1. Additional Fields (Type-Specific, `typeField1`–`4`)

Up to 4 free-text fields per inventory type, configured on the `inventorytype` record. Labels are set per type (e.g. "Engine Family", "Core Charge Code") and stored in `inventory.typedata1`–`4`. These are simpler than Q&A fields — plain text only, no choice lists, no typed values.

**When to look here:** The user asks for a field that sounds like it might vary by part type (e.g. "what's the engine family on this part?") or the organization calls them "additional fields" or "more labels."

```graphql
# Get the field labels for a specific inventory type
inventoryType(id: $typeId) {
  id
  name
  typeFieldLabel1
  typeFieldLabel2
  typeFieldLabel3
  typeFieldLabel4
}

# Get the field values on an inventory record
inventory(id: $inventoryId) {
  id
  description
  typeField1
  typeField2
  typeField3
  typeField4
}
```

To display these meaningfully, fetch the `inventoryType` labels alongside the inventory record values.

### 2. Custom Fields / Q&A Fields (`optionValues`, `inventoryOptions`)

Richer system — supports choice lists, typed values (number, date, boolean, currency), public flag, and filtering via `InventoryFieldFilter`. Labels can apply across all inventory types or be scoped to specific types — check `inventoryOptions` to see which fields exist and how they're configured. See Schema section below for full documentation.

**When to look here:** The user asks for a field that's consistent across part types, or uses terms like "custom fields", "Q&A fields", or "additional info."

**Decision rule:** If unsure which system a field lives in, check `inventoryOptions` first (Q&A definitions are a flat list and cheap to fetch), then check `inventoryType.typeFieldLabel1`–`4` for the relevant type. Present whichever has a matching label.

---



### Customers
```graphql
# Fetch all available Custom Field definitions for customers
query GetCustomerOptions {
  customerOptions {
    id
    label
    dataType
    showInCustomerList
  }
}

# Custom Field values live on the Customer type
type CustomerOptionValue {
  option: CustomerOption!   # contains id, label, dataType
  value: String!
}

# Access via:
customer(id: $id) {
  optionValues {
    option { id label }
    value
  }
}
```

### Vendors
```graphql
# Fetch all available Custom Field definitions for vendors
query GetVendorOptions {
  vendorOptions {
    id
    name
    type
    showInList
  }
}

# Custom Field values on the Vendor type
type VendorOptionValue {
  option: VendorOption!   # contains id, name, type
  value: String!
}

# Access via:
vendor(id: $id) {
  optionValues {
    option { id name }
    value
  }
}
```

### Inventory / Vehicles
```graphql
# Fetch Custom Field definitions — NOTE: inventoryOptions is a flat list, not paginated.
# Do NOT add pagination arguments — it will throw a schema error.
query GetInventoryOptions {
  inventoryOptions {
    id
    name
    dataType
    rank
    required
  }
}

# Custom Field values on Inventory and Vehicle types
type InventoryOptionValue {
  id: Int!
  optionId: Int!
  option: InventoryOption!
  value: String
}

# Access via:
inventory(id: $id) {
  optionValues {
    option { id name }
    value
  }
}

vehicle(id: $id) {
  optionValues {
    option { id name }
    value
  }
}
```

---

## Query Rules

### Auto-Detection: When to Query Custom Fields

Automatically include Custom Fields in your response (without the user asking) when:

- User asks to "show all info" or "full details" about a customer, vendor, inventory part, or vehicle
- User asks about a field that doesn't exist on the standard record type
- User mentions "Q&A fields", "custom fields", "flex fields", "additional fields", or "more labels" explicitly
- The question implies database-specific data (e.g., industry-specific fields unique to their setup)

Always look up `customerOptions` / `vendorOptions` / `inventoryOptions` first to understand
what Custom Fields exist in this user's database before claiming a field doesn't exist.
For inventory parts, also check `inventoryType.typeFieldLabel1`–`4` — the field may be a
type-specific Additional Field rather than a Q&A Custom Field.

### Query Optimization Rules

Custom Field data can be very large. Follow these rules to keep queries fast and avoid
exhausting context window tokens.

#### Rule 1: Never fetch Custom Fields across a list — always fetch by specific record

❌ **Bad** — fetches all records just to get fields:
```graphql
customers(pagination: { pageNumber: 1, pageSize: 1000 }) {
  items {
    id
    companyName
    optionValues { option { label } value }
  }
}
```

✅ **Good** — fetch the specific record, then its Custom Fields:
```graphql
customer(id: 123) {
  id
  companyName
  optionValues {
    option { id label }
    value
  }
}
```

#### Rule 2: Fetch field definitions once, then filter client-side

When you need to find which field label matches a user's request (e.g., "Fleet Size"):

```graphql
# Step 1: Get all field definitions (cheap — small list)
query GetCustomerFieldDefinitions {
  customerOptions {
    id
    label
    dataType
  }
}

# Step 2: Find the matching option ID client-side (no extra DB query)
# Step 3: When fetching customer records, only extract that field's value
```

#### Rule 3: When scanning multiple records — paginate tightly

Use small page sizes and filter on standard fields first to reduce the working set.

```graphql
customers(
  pagination: { pageNumber: 1, pageSize: 50 }
  filter: { active: true }
  orderBy: [{ field: "companyName", direction: ASC }]
) {
  items {
    id
    companyName
    optionValues {
      option { id label }
      value
    }
  }
  totalItems
  pageInfo { pageNumber pageSize totalPages }
}
```

#### Rule 4: Request only the fields you need

When you know which Custom Field the user wants, filter client-side after fetching —
don't request optionValues from hundreds of records unless you need them all.

### Token Exhaustion Warning

Some databases have 100+ Custom Fields per entity. When combined with large record lists,
fetching `optionValues` across many records can exhaust context window tokens and cause
the query to fail.

**High-risk patterns to avoid:**
- Fetching `optionValues` inside a list query with large `pageSize`
- Requesting all customers + all their Custom Fields in one pass
- Nesting `optionValues` inside deeply paginated loops without limiting scope

**Safe patterns:**
- Fetch field definitions (`customerOptions`) once — they're small
- Fetch `optionValues` only on individual records when the user asks about a specific entity
- When scanning across records, use `pageSize: 25–50` max and inform the user of progress
- If the user asks for bulk Custom Field data, warn them first and suggest alternatives

**When a user asks for bulk Custom Field data:**
> "Fetching Custom Fields for all customers can be a large amount of data. To avoid
> hitting limits, would you like to narrow this to specific fields, or a specific
> group of customers?"

### Searching / Filtering by Custom Field Value

Custom Field filtering is supported **server-side** on `CustomerFilter`, `VendorFilter`, and `InventoryFilter` via the `customFields` argument. This is the correct and preferred approach.

#### Filter input shapes

**Customers and Vendors** use `CustomFieldFilter`:
```graphql
input CustomFieldFilter {
  name: String!      # matches the Custom Field label — case-insensitive
  value: StringFilter!
}

input StringFilter {
  eq: String   # exact match (case-insensitive)
  ne: String   # not equal (use ne: "" to find records where field is set)
  lt: String
  gt: String
  lte: String
  gte: String
}
```

**Inventory** uses `InventoryFieldFilter` — identical shape, different type name:
```graphql
input InventoryFieldFilter {
  name: String!
  value: StringFilter!
}
```

#### Examples

```graphql
# Find all customers where Language = "French" (case-insensitive)
customers(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: {
    active: true
    customFields: [{ name: "Language", value: { eq: "French" } }]
  }
  orderBy: [{ field: "companyName", direction: ASC }]
) {
  totalItems
  items { id companyName optionValues { option { label } value } }
}

# Find all customers where Language is set to any value
customers(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: {
    customFields: [{ name: "Language", value: { ne: "" } }]
  }
) { totalItems items { id companyName } }

# Filter by multiple Custom Fields (AND logic)
customers(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: {
    customFields: [
      { name: "Language", value: { eq: "French" } }
      { name: "Source", value: { eq: "web" } }
    ]
  }
) { totalItems items { id companyName } }

# Inventory — filter by a custom field value
inventories(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: {
    stores: [1]
    statuses: [A]
    customFields: [{ name: "engine", value: { ne: "" } }]
  }
) { totalItems items { id description } }

# Vendor — filter by custom field
vendors(
  pagination: { pageNumber: 1, pageSize: 100 }
  filter: {
    customFields: [{ name: "Is in use", value: { eq: "true" } }]
  }
) { totalItems items { id companyName } }
```

#### Behavior notes

- **Case-insensitive**: `eq: "french"`, `eq: "French"`, and `eq: "FRENCH"` return identical results.
- **Multiple filters = AND**: All `customFields` entries must match for a record to be included.
- **`ne: ""`** is the idiomatic way to find records where a field has any value set.
- **All values are strings**: Custom Field values are stored as strings regardless of their `dataType`. This means `gt`/`lt`/`gte`/`lte` comparisons are **lexicographic, not numeric** — e.g. filtering `value: { gt: "9" }` would place `"10"` before `"9"` alphabetically and produce incorrect results for numeric fields. Proper numeric server-side filtering is planned for a future API version. For now, use `eq`/`ne` server-side and handle any numeric range logic client-side (see Fallback below).
- **Boolean fields**: Stored values may be inconsistently cased (`"true"` vs `"True"`). Case-insensitive matching means `eq: "true"` will catch both.
- **`name` matches by label** (the human-readable field name), not by option ID. If two fields share the same name, both will be matched — use `inventoryOptions` to check for duplicate names.
- **Pagination still applies**: server-side filtering reduces the result set, but you still need to paginate if `totalItems > pageSize`. Follow the enterprise-query skill pagination rules.

#### Fallback: client-side filtering

Use this when the server does not support `customFields` filtering (see Common Mistakes), or when the query requires **numeric range logic** — since all Custom Field values are stored as strings, server-side `gt`/`lt` comparisons are lexicographic and will produce incorrect results for number-typed fields.

For example, "find all engines with mileage over 10,000 miles" cannot be reliably handled server-side. Instead, fetch the records and filter in code:

```
matchingInventory = []
page = 1

loop:
  result = query inventories(pageNumber: page, pageSize: 50, filter: { stores: [1], statuses: [A] })

  for each item in result.items:
    mileageValue = item.optionValues
      .find(ov => ov.option.name == "Mileage")?.value

    if mileageValue && int(mileageValue) > 10000:
      matchingInventory.append(item)

  if page >= result.pageInfo.totalPages: break
  page += 1
```

### Common Mistakes

| Mistake | Fix |
|---|---|
| Saying "optionsValues table" to the user | Say "Custom Fields" |
| Fetching all records to find one Custom Field value | Use `customFields` filter in `CustomerFilter` / `VendorFilter` / `InventoryFilter` server-side |
| Not querying Custom Fields when user asks for unlisted data | Always check — it's probably a Custom Field or an Additional Field |
| Showing null/empty Custom Fields | Show "Not set" for empty values |
| Forgetting to fetch field definitions before matching user labels | Run `customerOptions` / `vendorOptions` / `inventoryOptions` first |
| Using <code>customFields</code> filter and getting a schema error | The server may be on an older API version. Inform the user that a newer version is likely available and they should contact ITrack Support — then fall back to client-side pagination in the meantime. |
| Adding `pagination` args to `inventoryOptions` | `inventoryOptions` is a flat list — no pagination arguments supported |
| Using numeric comparison operators on number-typed Custom Fields | Values are stored as strings — numeric GT/LT comparisons are lexicographic, not numeric. Do numeric comparisons client-side. |
| Expecting `customFields` filter to match by option ID | The `name` field matches by **label** string, not ID |
| Only checking `inventoryOptions` for inventory part fields | Inventory parts also have `typeField1`–`4` (Additional Fields) configured per inventory type — check both systems |

---

## Workflows & Patterns

### Answering a Specific Custom Field Question

1. Fetch field definitions: `customerOptions`, `vendorOptions`, or `inventoryOptions`
2. Match the user's label to an option ID client-side
3. Note the field's `dataType` (text, number, date, boolean, etc.) for display formatting
4. Fetch the specific record by ID with `optionValues` included
5. Filter to the relevant field(s) and present the value — show "Not set" if empty

```graphql
query GetCustomerWithCustomFields($id: UInt!) {
  customer(id: $id) {
    id
    companyName
    contactName
    optionValues {
      option { id label dataType }
      value
    }
  }
}
```

### Full Customer Profile

When a user asks to "show everything" about a customer, include Custom Fields in a single query:

```graphql
query FullCustomerProfile($id: UInt!) {
  customer(id: $id) {
    id
    companyName
    contactName
    phoneNumber
    email
    active
    notes
    optionValues {
      option { label dataType }
      value
    }
  }
}
```

Render Custom Fields after standard fields, grouped under a "Custom Fields" heading.
Show "Not set" for empty values — don't hide or null them.

### Full Vendor Profile

```graphql
query FullVendorProfile($id: UInt!) {
  vendor(id: $id) {
    id
    companyName
    contactName
    phoneNumber
    email
    optionValues {
      option { name type }
      value
    }
  }
}
```
