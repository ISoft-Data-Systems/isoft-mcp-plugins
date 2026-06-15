# Out-of-Spec Query Reference

## Variables Reference

```
DateTime format: "YYYY-MM-DDT00:00:00Z" (start of day) / "YYYY-MM-DDT23:59:59Z" (end of day)
pageSize: 120 is a good balance for samples — large enough to reduce round trips, small
          enough to stay within context limits. Max is 500.
```

## Query 1: Out-of-Spec Samples — Single or Multi-Plant

```graphql
query {
  samples(
    filter: {
      plantIds: [<id1>, <id2>]         # all target plants in one call
      performedFrom: "YYYY-MM-DDT00:00:00Z"
      performedTo: "YYYY-MM-DDT23:59:59Z"
    }
    pagination: { pageNumber: 1, pageSize: 120 }
    orderBy: [performed_DESC]
  ) {
    data {
      id
      tagNumber
      performed
      plant { id name code }
      analysis { id name category analysisType }
      sampleValues {
        id
        result
        resultStatus
        filledOut
        analysisOption { option unit valueType }
      }
    }
    info { totalItems totalPages pageNumber }
  }
}
```

After fetching: filter where `sampleValues[].resultStatus != "ALLOWED"`.

## Query 2: Subsequent Pages

```graphql
# Same query, increment pageNumber:
pagination: { pageNumber: 2, pageSize: 120 }
# Repeat until pageNumber >= totalPages
```

## Query 3: Using get_blank_values for Missing Results

The `get_blank_values` MCP tool handles unfilled/incomplete results natively.
Use it instead of a custom query when the user asks about blank or missing entries.

```
Tool: get_blank_values
Parameters:
  plantIds: [<ids>]              # required
  modifiedFrom: "YYYY-MM-DDT..."  # optional date range
  modifiedTo: "YYYY-MM-DDT..."
  pageNumber: 1
  pageSize: 100
```

## ResultStatus Values (complete enum)

| Value | Meaning | Severity |
|-------|---------|----------|
| OUT_OF_BOUNDS | Exceeds defined limits | Critical |
| ERROR | Calculation or system error | Critical |
| WARNING | Advisory range exceeded | Moderate |
| ALLOWED | Passing | None |
| NOT_CALCULATED | No threshold applied | Informational |

## Client-Side Filtering Pattern

```python
# After fetching samples:
out_of_spec = []
for sample in results:
    bad_values = [
        sv for sv in sample['sampleValues']
        if sv['resultStatus'] in ('OUT_OF_BOUNDS', 'ERROR')
    ]
    if bad_values:
        out_of_spec.append({
            'sample_id': sample['id'],
            'tag': sample['tagNumber'],
            'plant': sample['plant']['code'],
            'analysis': sample['analysis']['name'],
            'issues': bad_values
        })
```
