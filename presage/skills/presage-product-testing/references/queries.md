# Product Testing Query Reference

## Query 1: Find Product by Name Keyword

```graphql
query {
  products(
    filter: {
      plantIds: [<id>]
      nameContains: "<keyword>"
      activeOnly: false
    }
    pagination: { pageNumber: 1, pageSize: 30 }
    orderBy: [NAME_ASC]
  ) {
    data {
      id
      name
      category
      productType
      active
      inUseAtPlants { id name code }
    }
    info { totalItems totalPages }
  }
}
```

## Query 2: All Products at a Plant (Browse Mode)

```graphql
query {
  products(
    filter: { plantIds: [<id>], activeOnly: true }
    pagination: { pageNumber: 1, pageSize: 100 }
    orderBy: [NAME_ASC]
  ) {
    data { id name productType category active }
    info { totalItems totalPages }
  }
}
```

## Query 3: Product with Batch / Lot Info

```graphql
query {
  products(
    filter: { plantIds: [<id>], nameContains: "<keyword>" }
    pagination: { pageNumber: 1, pageSize: 10 }
  ) {
    data {
      id name active
      productBatches {
        id
        lotNumber
        status
        expirationDate
      }
    }
    info { totalItems }
  }
}
```

## Query 4: Samples for Finished Product Testing

After identifying the product/plant, get samples and filter by analysis name client-side:

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
      workOrder { id title }
      sampleValues {
        result resultStatus filledOut
        analysisOption { option unit }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

Filter client-side for the analysis name(s) that correspond to the product test type
the user is asking about (e.g., "Finished Product", "Grab Sample", "Water Quality").

## ProductFilter Fields

```
plantIds: [Int]          — Plants to search within
nameContains: String     — Partial name match, case-insensitive
activeOnly: Boolean      — false = include discontinued
productType: Enum        — INGREDIENT or FINISHED_PRODUCT (optional)
```

## Product Batch Status Values

| Status | Meaning |
|--------|---------|
| ACTIVE | Currently in use |
| QUARANTINE | On hold pending review |
| RELEASED | Cleared for use after hold |
| REJECTED | Failed QC |
| EXPIRED | Past expiration date |
