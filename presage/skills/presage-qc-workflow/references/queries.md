# QC Workflow Query Reference

## Query 1: Open Work Orders at a Plant

```graphql
query {
  workOrders(
    filter: {
      plantId: <Int>              # singular — one plant per call
      statuses: [OPEN, SAMPLED]
      from: "YYYY-MM-DD"
      to: "YYYY-MM-DD"
    }
    pagination: { pageNumber: 1, pageSize: 50 }
    orderBy: { columnName: "dateCreated", direction: "DESC" }
  ) {
    data {
      id
      title
      documentStatus
      scheduled
      dateCreated
      plant { id name code }
      samples {
        id tagNumber status
        analysis { id name category }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

## Query 2: All Work Orders for Audit (Including Closed)

```graphql
query {
  workOrders(
    filter: {
      plantId: <Int>
      statuses: [OPEN, SAMPLED, CLOSED]
      from: "YYYY-MM-DD"
      to: "YYYY-MM-DD"
    }
    pagination: { pageNumber: 1, pageSize: 50 }
    orderBy: { columnName: "dateCreated", direction: "ASC" }
  ) {
    data {
      id title documentStatus scheduled dateCreated
      samples {
        id tagNumber status
        analysis { name category }
        sampleValues {
          result resultStatus filledOut
          analysisOption { option unit }
        }
      }
    }
    info { totalItems totalPages }
  }
}
```

## Query 3: Calibration Samples

Step 1 — Find calibration analysis (use presage-analysis-finder pattern):
```graphql
analyses(
  filter: { plantIds: [<id>], nameContains: "Calibration", activeOnly: false }
  pagination: { pageNumber: 1, pageSize: 20 }
) {
  data { id name category inUseAtPlants { id name code } }
  info { totalItems }
}
```

Step 2 — Get samples in date range:
```graphql
samples(
  filter: {
    plantIds: [<id>]
    performedFrom: "YYYY-MM-DDT00:00:00Z"
    performedTo: "YYYY-MM-DDT23:59:59Z"
  }
  pagination: { pageNumber: 1, pageSize: 120 }
  orderBy: [performed_DESC]
) {
  data {
    id tagNumber performed status
    analysis { id name category }
    workOrder { id title documentStatus }
    sampleValues { result resultStatus filledOut analysisOption { option unit } }
  }
  info { totalItems totalPages }
}
```

Filter client-side: keep samples where `analysis.name` contains the calibration keyword.

## Using get_blank_values Tool

No GraphQL query needed — call the MCP tool directly:

```
Tool: get_blank_values
Parameters:
  plantIds: [<ids>]              required
  modifiedFrom: "YYYY-MM-DDT..."  optional
  modifiedTo: "YYYY-MM-DDT..."    optional
  optionName: "<string>"          optional — filter to specific parameter
  pageNumber: 1
  pageSize: 100
```

Returns SampleValue records where `filledOut` is null.

## Work Order Status Values

| Status | Description |
|--------|-------------|
| OPEN | Created, not yet started |
| SAMPLED | In progress |
| CLOSED | Complete |
| CANCELLED | Voided |

## Sample Status Values

| Status | Description |
|--------|-------------|
| OPEN | No results yet |
| SAMPLED | Results being entered |
| CLOSED | Complete |
| CANCELLED | Voided |

## Multi-Plant Work Order (Same Session, No Re-Auth)

```
# Call once per plant — same session, do NOT close/re-authenticate:
workOrders(filter: { plantId: <plant_A_id>, ... })
workOrders(filter: { plantId: <plant_B_id>, ... })
workOrders(filter: { plantId: <plant_C_id>, ... })
```
