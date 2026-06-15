# Analysis Finder Query Reference

## Query 1: Find Analysis by Keyword + Which Plants Use It

```graphql
query {
  analyses(
    filter: {
      plantIds: [<any plant ID>]
      nameContains: "<keyword>"
      activeOnly: false
    }
    pagination: { pageNumber: 1, pageSize: 50 }
    orderBy: [NAME_ASC]
  ) {
    data {
      id
      name
      category
      analysisType
      inUseAtPlants { id name code }
    }
    info { totalItems totalPages }
  }
}
```

## Query 2: Browse All Analyses at a Plant

When no specific analysis name is known and you need to find what's available:

```graphql
query {
  analyses(
    filter: { plantIds: [<id>], activeOnly: true }
    pagination: { pageNumber: 1, pageSize: 100 }
    orderBy: [NAME_ASC]
  ) {
    data {
      id
      name
      category
      analysisType
    }
    info { totalItems totalPages }
  }
}
```

Paginate if `totalPages > 1`. Scan names for the closest match to what the user described.

## Query 3: Samples Filtered to a Named Analysis (Client-Side)

```graphql
query {
  samples(
    filter: {
      plantIds: [<ids>]
      performedFrom: "YYYY-MM-DDT00:00:00Z"
      performedTo: "YYYY-MM-DDT23:59:59Z"
    }
    pagination: { pageNumber: 1, pageSize: 120 }
    orderBy: [performed_DESC]
  ) {
    data {
      id tagNumber performed
      plant { id name code }
      analysis { id name category }
      sampleValues {
        result resultStatus
        analysisOption { option unit }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

Filter client-side: keep samples where `analysis.name` contains your target keyword.

## AnalysisOption Field Names (Gotcha)

| Wrong | Right |
|-------|-------|
| `name` | `option` |
| `dataType` | `valueType` |

## Analysis Type Values

| Value | Meaning |
|-------|---------|
| `TESTING` | Standard lab analysis / QC test |
| `RECIPE` | Production batch / manufacturing process |
