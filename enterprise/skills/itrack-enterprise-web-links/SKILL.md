---
name: itrack-enterprise-web-links
description: >
  Generate clickable deep-links to records in ITrack Enterprise Web Edition. Use this skill
  whenever the user wants a link, URL, or clickable reference to any ITrack record type —
  parts, vehicles, customers, vendors, sales orders, dashboard reports, or any other record type. Triggers include:
  "link to [record type] [id]", "give me a URL for this record", "can you link to that record",
  "make that id clickable", "generate a link to this record", or any time a record ID appears in
  results and the user would benefit from a direct link. Also use proactively when returning
  query results that include record IDs — include links alongside the data so the user can
  jump directly to the record.
metadata:
  author: ISoft Data Systems, Inc.
  version: 0.2.0
  mcp-server: Enterprise-Api-Extension
  category: universal
---

# ITrack Enterprise Web Edition — Deep Links

## Base URL

> ⚠️ **NEVER guess or invent the domain.** Always derive it programmatically.

Call `get_session_info` to retrieve the GraphQL host:

```
GraphQL Host: https://your-company.itrackenterprise.com/graphql
```

Strip `/graphql` to get the web UI base URL:

- `https://your-company.itrackenterprise.com/graphql` → `https://your-company.itrackenterprise.com`

If `get_session_info` has not been called yet in the current conversation, call it before generating any links. Do not proceed with link generation until the host is confirmed.

---

## Display Guidelines

Format links based on context — use judgment:
- **Plain response**: present the full URL as-is
- **Artifact / HTML table**: hyperlink on the record ID (or name if ID isn't shown)
- **Proactive**: when returning query results that include record IDs, include links without being asked

> The Enterprise web interface is actively being developed. Link patterns will expand as more
> record types become available.

---

## Store Context

Parts and vehicles are inventory records assigned to a specific store representing their
physical location. Always include the correct `storeId` for these records. In multi-store
organizations, a user may be able to view inventory at another store but not edit it — the
correct `storeId` ensures the user lands in the right store context for their permissions.

Non-inventory records (customers, vendors) are not store-specific and do not require a
`storeId` in their URLs.

---

## URL Patterns

Only specify a tab when context calls for a direct link to a specific view. The first tab
listed for each record type is the default. Note that tab URL values do not always match their
UI labels.

### Part

```
https://<domain>/#/part/<storeId>/<partId>
```

- `storeId` — in path; see Store Context above
- `partId` — in path

To deep-link to a specific tab, append `?tab=<tab>`:

| UI Label | URL Value |
|----------|-----------|
| Basic *(default)* | `basic` |
| Attachments | `attachments` |
| History | `history` |
| Links | `links` |

---

### Vehicle

Default (no specific tab):
```
https://<domain>/#/vehicle?storeId=<storeId>&vehicleId=<vehicleId>
```

Specific tab (tab name is in the **path**, not a query parameter):
```
https://<domain>/#/vehicle/<tab>?storeId=<storeId>&vehicleId=<vehicleId>
```

- `vehicleId` — in query string (required)
- `storeId` — in query string; see Store Context above. The record loads without it, but
  include when known.

| UI Label | URL Value |
|----------|-----------|
| Basic *(default)* | `basic` |
| Attachments | `attachment` |
| History | `history` |
| Teardown | `teardown` |
| Break Even Graph | `break-even` |

---

### Sales Order

```
https://<domain>/#/sales-order/<storeId>/<salesOrderId>
```

- `storeId` — in path
- `salesOrderId` — in path

The sales order screen has two independent tabbed areas. Both are optional — omit to load the default view.

**Top tabs** (`selectedCustomerDetailTab`): customer and document-level info

| UI Label | URL Value |
|----------|-----------|
| Basic Info *(default)* | `CUSTOMER_DETAIL` |
| Billing And Shipping | `BILLING_AND_SHIPPING` |

**Bottom tabs** (`selectedTab`): line item and transaction details

| UI Label | URL Value |
|----------|-----------|
| Line Items *(default)* | `LINE_ITEM` |
| Adjustments | `ADJUSTMENTS` |
| Attachments | `ATTACHMENTS` |
| Messages | `MESSAGES` |
| Payments | `PAYMENTS` |

To deep-link to specific tabs, append both params:
```
https://<domain>/#/sales-order/<storeId>/<salesOrderId>?selectedCustomerDetailTab=<top-tab>&selectedTab=<bottom-tab>
```

---

### Dashboard Report

```
https://<domain>/#/dashboard/<reportId>
```

- `reportId` — integer dashboard report ID
- Not store-specific

Dashboard reports are containers of charts. Before linking a user to a report, query the API
to confirm the report is accessible to them and that its charts are relevant to their question.
Key fields to check:

- `sharedWithCurrentUser` — whether the report is visible to the current user
- `shareType` — sharing scope: `EVERYONE`, `GROUP`, `STORE`, or `USER` (private)
- `charts` — list of charts on the report; check `title` and `type` to confirm relevance

Linking to a dashboard report is most useful when the user wants to explore data interactively
or monitor metrics on an ongoing basis, rather than receive a one-time query result in chat.

---

### Report Viewer

```
https://<domain>/#/report-viewer/reports/<reportId>
```

- `reportId` — integer report ID
- Not store-specific
- Query string parameters (e.g. `?selectedCategory=Sales`) are optional and not required for the report to load

Report Viewer reports are distinct from Dashboard Reports. Users typically interact with them via print or print preview rather than interactively. Use this link format when directing a user to a saved report in the Reports section of the UI.

---

### Customer

```
https://<domain>/#/customer/<customerId>
```

- `customerId` — in path
- Not store-specific

To deep-link to a specific tab, append `?tab=<tab>`:

| UI Label | URL Value |
|----------|-----------|
| Address Info *(default)* | `address` |
| Additional Info | `additional` |
| Attachments | `attachments` |
| Payment/Tax Info | `paymentAndTax` |
| Sales | `sales` |

---

### Vendor

```
https://<domain>/#/vendor/<vendorId>
```

- `vendorId` — in path
- Not store-specific

To deep-link to a specific tab, append `?tab=<tab>`:

| UI Label | URL Value |
|----------|-----------|
| Address Info *(default)* | `address` |
| Additional Info | `additional` |
| Attachments | `attachments` |
| Purchase Orders | `purchaseOrders` |

---

### Home Screen

The home screen is the starting view for new sessions and the default destination when no
specific record or view is known.

```
https://<domain>/#/home
```

The home screen displays a document list tabbed by document type. Available tabs:

| Tab | Path segment |
|-----|-------------|
| Sales Orders | `sales-order` |
| Purchase Orders | `purchase-order` |
| Work Orders | `work-order` |
| Transfer Orders | `transfer-order` |

To link to a specific tab:
```
https://<domain>/#/home/<tab>
```

User default filter preferences are stored in user settings and applied automatically on
load — omit parameters unless the user wants to override their defaults. When a user asks a
question that maps to a filterable list (e.g. "show me sales orders this month"), consider
offering a filtered home screen link as a complement to or alternative to querying the data
directly — especially for open-ended browsing requests where the user may want to explore
results themselves in the UI.

Dates use `YYYY-MM-DD` format. All parameters are optional.

**Sales Orders** — `/#/home/sales-order`
**Work Orders** — `/#/home/work-order`

Document types for both are org-defined and control document behavior (e.g. whether inventory
is decremented, whether it is a customer-facing or internal document). Users are typically more
familiar with their sales order document type names and tend to have more of them; work order
type names may be less familiar. If unsure of the exact name, query the API to confirm before
constructing a `documentType` filter.

| Parameter | Description |
|-----------|-------------|
| `documentType` | Filter by doc type (org-defined, URL-encode spaces, e.g. `Price%20Quote`) |
| `fromDate` | Start of date range |
| `toDate` | End of date range |
| `showClosed` | Include closed orders (`true`/`false`) |
| `storeId` | Filter by store |
| `userId` | Filter by user |

Work Orders also support:

| Parameter | Description |
|-----------|-------------|
| `showEstimates` | Include estimate work orders (`true`/`false`) |

**Purchase Orders** — `/#/home/purchase-order`
**Transfer Orders** — `/#/home/transfer-order`

No `documentType` filter. Available parameters:

| Parameter | Description |
|-----------|-------------|
| `fromDate` | Start of date range |
| `toDate` | End of date range |
| `showDoneReceiving` | Include orders with receiving complete (`true`/`false`) |
| `showPricesFinalized` | Include orders with prices finalized (`true`/`false`) |
| `storeId` | Filter by store |
| `userId` | Filter by user |

---

## Expanding This Skill

When adding a new record type:
1. Confirm the URL pattern in the browser.
2. Document the route, required parameters, and tab values (note any mismatches between UI labels and URL values).
3. Add the record type to the skill description triggers.

**Intentionally deferred:**
- Part search (`/#/part-search/results`) and vehicle search (`/#/vehicle-search/results`) screens
  support URL-based filters but the parameter structure is complex (URL-encoded JSON, array
  values, inventory type codes, custom field maps, etc.). Document these fully before adding,
  including how to correctly construct each parameter.
