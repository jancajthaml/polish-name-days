# ADR-01-02: Data Schema — JSON Name-to-Dates Map

## Context

The current data format is CSV (`Name,DD.MM`) stored in a flat file. Firebase Realtime Database (chosen in [ADR-01-01](adr-01-01-realtime-data-service.md)) stores data as a JSON tree. The topology eliminated data format as a separate concern — the format follows the data service's native representation. PRD-002 requires entries to conform to a schema (non-empty name, valid DD.MM date). PRD-003 requires the search to operate on the same semantic structure (name → list of dates).

See also: [PRD-002](../../prd/prd-002-data-management.md), [PRD-003](../../prd/prd-003-search-with-dynamic-data.md)

## Decision

We choose a **flat JSON map** structure in Firebase RTDB where each key is a Polish name and each value is an object of date entries:

```json
{
  "nameDays": {
    "Zofia": {
      "0": "15.05",
      "1": "02.09",
      "2": "18.09",
      "3": "23.09",
      "4": "30.09",
      "5": "18.12"
    },
    "Wojciech": {
      "0": "23.04"
    }
  }
}
```

Firebase RTDB does not support native arrays — it uses ordered integer-keyed objects as the array equivalent. This is the standard RTDB pattern for list data.

### Schema rationale

- **Name as key**: Mirrors the current in-memory structure (`{ name: [date, ...] }`). Enables O(1) lookup by name. Prevents duplicate name entries (a name key either exists or it doesn't).
- **Dates as indexed sub-object**: Each date is stored under an integer key. Adding a new date for an existing name appends to the sub-object. Firebase RTDB efficiently synchronises sub-key changes.
- **Flat structure**: One level deep under `/nameDays`. No nested collections, no relational joins. Firebase RTDB performs best with shallow trees.

### Firebase Security Rules with validation

```json
{
  "rules": {
    "nameDays": {
      ".read": true,
      ".write": "auth != null",
      "$name": {
        ".validate": "newData.hasChildren()",
        "$index": {
          ".validate": "newData.isString() && newData.val().matches(/^[0-3][0-9]\\.[0-1][0-9]$/)"
        }
      }
    }
  }
}
```

### Client-side data transformation

The Firebase SDK delivers this JSON to the client. The client transforms it to the existing in-memory format:

```javascript
// Firebase delivers: { "Zofia": { "0": "15.05", "1": "02.09" }, ... }
// Client transforms to: { "Zofia": ["15.05", "02.09"], ... }
const data = {};
const snapshot = /* from onValue callback */;
snapshot.forEach((child) => {
  data[child.key] = Object.values(child.val());
});
```

This transformation is trivial and preserves the exact data shape the search algorithm expects.

### Alternatives considered

| Alternative | Why rejected |
|------------|-------------|
| Array values (`"Zofia": ["15.05", "02.09"]`) | Firebase RTDB converts arrays to indexed objects internally. Writing `["15.05"]` stores as `{"0": "15.05"}`. Using the indexed-object form explicitly avoids surprise behaviour on reads and partial updates. |
| Flat records (`[{ name: "Zofia", date: "15.05" }, ...]`) | Creates one record per name-date pair (~6600 records). Firebase RTDB synchronises the entire list on initial load regardless. The grouped-by-name structure is more compact and matches the search algorithm's expected input. |
| Preserve CSV in a single RTDB string node | Would require client-side CSV parsing, negating the benefit of structured data. Firebase listeners would push the entire CSV string on any change, no incremental updates possible. |

## Failure modes and mitigations

| Failure mode | Mitigation |
|-------------|-----------|
| Operator writes a name with no dates (empty sub-object) | Firebase `.validate` rule requires `newData.hasChildren()` on name nodes. Write is rejected. Client-side validation also skips entries with zero dates. |
| Operator writes an invalid date format (e.g., "32.13") | Firebase `.validate` rule enforces DD.MM regex pattern. Client-side validation uses the same regex as a secondary check. |
| Name key contains characters invalid for Firebase RTDB keys (`.`, `#`, `$`, `[`, `]`) | Polish names do not contain these characters. If an edge case arises, the write is rejected by Firebase. No mitigation needed beyond Firebase's built-in key validation. |
| Large batch import causes temporary spike in download bandwidth | One-time concern during initial CSV-to-RTDB migration. The entire dataset is ~200 KB — well within free-tier limits even with many simultaneous listeners. |

## Consequences

- The CSV file (`name-days.csv`) is no longer the authoritative data source. It may be retained in the repository as a historical reference or static fallback.
- The client-side CSV parser (`data.split`, `line.split(',')`) is replaced by a JSON-to-map transformation from the Firebase snapshot.
- Data management is done through Firebase Console or REST API using the JSON schema above.
- Adding a new name: write a new key under `/nameDays` with its dates sub-object.
- Adding a new date to an existing name: append a new indexed entry to the name's sub-object.
