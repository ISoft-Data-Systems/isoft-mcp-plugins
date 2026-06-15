# Presage API — Optimized Query Patterns

## Query Efficiency Rules

Always follow these rules before writing any query:

1. **Use date filters** — Never query without `from`/`to` or `performedFrom`/`performedTo`. Unbounded queries return the entire history.
2. **Set pagination** — Always include `pagination: { pageNumber: 1, pageSize: N }`. Start with 10–25 for exploration, up to 120 for sample bulk reads.
3. **Request only needed fields** — Do not select every field. Minimal field selection = smaller response = safer context window.
4. **Never loop through all plants** — Use multi-plant filter arrays (`plantIds: [1, 2, 3]`) or the `inUseAtPlants` pattern below.
5. **Verify with a small query first** — Use `pageSize: 1` or `pageSize: 5` to confirm response shape before scaling up.

---

## Work Orders by Date Range

```graphql
query {
  workOrders(
    filter: {
      from: "YYYY-MM-DD"
      to: "YYYY-MM-DD"
      plantId: 1
      statuses: [OPEN, SAMPLED]
    }
    pagination: { pageNumber: 1, pageSize: 25 }
    orderBy: { columnName: "dateCreated", direction: "DESC" }
  ) {
    data {
      id
      title
      documentStatus
      scheduled
      dateCreated
      plant { name }
      samples {
        id
        tagNumber
        status
      }
    }
    info {
      totalItems
      pageNumber
      totalPages
    }
  }
}
```

---

## Samples with Test Results

```graphql
query {
  samples(
    filter: {
      plantIds: [1]
      statuses: [OPEN, SAMPLED]
      performedFrom: "YYYY-MM-DDT00:00:00Z"
      performedTo: "YYYY-MM-DDT23:59:59Z"
    }
    pagination: { pageNumber: 1, pageSize: 50 }
    orderBy: [tagNumber_ASC]
  ) {
    data {
      id
      tagNumber
      status
      created
      performed
      analysis {
        name
        category
        analysisType
      }
      sampleValues {
        id
        result
        resultStatus
        analysisOption {
          option
          valueType
          unit
        }
      }
    }
    info {
      totalItems
      pageNumber
      totalPages
    }
  }
}
```

---

## Single Sample with Full Detail

Use this when drilling into a specific sample by ID.

```graphql
query {
  sample(id: 12345) {
    id
    tagNumber
    status
    created
    performed
    comments
    findings
    analysis {
      id
      name
      category
      analysisType
      options {
        option
        valueType
        unit
        requiredToClose
        requiredToPerform
        defaultValue
      }
    }
    sampleValues {
      id
      result
      resultStatus
      filledOut
      analysisOption {
        option
        valueType
        unit
      }
    }
    workOrder {
      id
      title
      documentStatus
    }
  }
}
```

---

## Out-of-Spec Results

Filter sample values by `resultStatus` to find failing results. Typical non-passing values include `FAIL`, `OUT_OF_SPEC`, or similar — verify exact enum values with `explore_schema(type: "enum", items: "ResultStatus")`.

```graphql
query {
  samples(
    filter: {
      plantIds: [1]
      performedFrom: "YYYY-MM-DDT00:00:00Z"
      performedTo: "YYYY-MM-DDT23:59:59Z"
    }
    pagination: { pageNumber: 1, pageSize: 50 }
  ) {
    data {
      id
      tagNumber
      sampleValues {
        result
        resultStatus
        analysisOption {
          option
        }
      }
    }
    info { totalItems }
  }
}
```

Then filter client-side for entries where `resultStatus` is not passing, or add a server-side `resultStatus` filter if supported by `SampleFilter`.

---

## Products by Plant with Batches

```graphql
query {
  products(
    filter: { plantIds: [1], activeOnly: true }
    pagination: { pageNumber: 1, pageSize: 25 }
    orderBy: [NAME_ASC]
  ) {
    data {
      id
      name
      productType
      category
      unit
      productBatches {
        id
        lotNumber
        status
        expirationDate
      }
    }
    info {
      totalItems
      pageNumber
    }
  }
}
```

---

## Plant/Analysis Matrix — Which Analyses Are Used Where

**Do NOT loop through each plant individually.** Use the `inUseAtPlants` field on `Analysis` to get a cross-plant view in a single query.

```graphql
query {
  analyses(
    pagination: { pageNumber: 1, pageSize: 50 }
  ) {
    data {
      id
      name
      analysisType
      category
      inUseAtPlants {
        id
        name
        code
      }
    }
    info {
      totalItems
      totalPages
    }
  }
}
```

If the schema does not have `inUseAtPlants`, verify with:
```
search_schema(keyword: "analysis")
explore_schema(type: "type", items: "Analysis")
```

---

## Work Order Status Counts

To get counts by status without pulling all records, request a small `pageSize` for each status and use `info.totalItems`:

```graphql
query OpenOrders {
  workOrders(
    filter: { plantId: 1, from: "YYYY-MM-DD", to: "YYYY-MM-DD", statuses: [OPEN] }
    pagination: { pageNumber: 1, pageSize: 1 }
  ) {
    info { totalItems }
  }
}
```

Run once per status (OPEN, SAMPLED, CLOSED) and summarize the `totalItems` counts. This avoids pulling full records just for a count.
