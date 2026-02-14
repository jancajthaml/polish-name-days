# ADR-02-02: Search Preservation — Client-side Levenshtein with Dynamic Index

## Context

The current application uses a custom Levenshtein distance implementation with `Intl.Collator("pl", { sensitivity: "base" })` for Polish diacritics-insensitive fuzzy matching. The algorithm scans all ~4700 name keys on every keystroke, sorting by distance and displaying the top 3 with distance < 5. The topology keeps search in the Browser Client. PRD-003 requires search to incorporate dynamically arriving names and maintain ≤200ms responsiveness at up to 2x scale (~10,000 names).

The Constraint Synthesis (01-constraints.md) assessed fuzzy search as "necessary — keep client-side; moving to server adds component and latency for no demonstrated need at ~4700 names."

See also: [PRD-003](../../prd/prd-003-search-with-dynamic-data.md)

## Decision

We choose to **preserve the existing Levenshtein algorithm and `Intl.Collator` unchanged**, with two modifications:

### 1. Dynamic key index

The `keys` array (currently `Object.keys(data)` computed once) becomes a mutable variable re-extracted on every `onValue` callback (see [ADR-02-01](adr-02-01-client-architecture.md)). The `reconsolidate` function references the live `keys` array. No change to the search algorithm itself.

### 2. Input-aware data-change handling

When new data arrives via `onValue`:
- If the search input is empty, no action beyond updating `data` and `keys`.
- If the search input is non-empty, re-invoke `reconsolidate` with the current input value. This makes newly arrived names immediately searchable and visible.

### Performance assessment

| Metric | Current (~4700 names) | Projected 2x (~10,000 names) |
|--------|-----------------------|-------------------------------|
| Levenshtein comparisons per keystroke | ~4700 | ~10,000 |
| Estimated time per keystroke (modern browser) | <50ms | <120ms |
| PRD-003 threshold | 200ms | 200ms |

The Levenshtein scan is O(n × m) where n = number of names and m = average name length (~7 characters). At 10,000 names, this is ~70,000 character comparisons — well within the 200ms budget on modern hardware. No debouncing, indexing, or server-side search is needed at this scale.

### What is NOT changed

- The Levenshtein algorithm implementation (lines 44-92 of current `index.html`).
- The `Intl.Collator("pl", { sensitivity: "base" })` configuration.
- The top-3-results-with-distance-<-5 display logic.
- The `keyup` event trigger.

### Alternatives considered

| Alternative | Why rejected |
|------------|-------------|
| Server-side search (Firebase Cloud Functions + full-text search) | Adds a component (Cloud Function), latency (network round-trip per keystroke), and cost. Not justified at current or projected scale. Violates topology (search stays in browser). |
| Pre-built search index (e.g., Fuse.js) | Adds a dependency (~25 KB). Fuse.js uses a different algorithm (bitap/extended search) that would change the search behaviour. The existing Levenshtein with Polish collator is proven correct and performant. |
| Debounced search | Not needed at current scale (<50ms per search). Could be added as a future optimisation if dataset grows beyond projections, but adding it now complicates the input UX (delayed results). |
| Web Worker for search | Moves Levenshtein computation off the main thread. Not needed — <50ms is below the 16ms frame budget concern threshold for UI jank. The overhead of postMessage serialisation would negate the benefit. |

## Failure modes and mitigations

| Failure mode | Mitigation |
|-------------|-----------|
| Dataset grows beyond performance threshold (FS-5) | Polish name days are cultural data with very low growth rate. If the dataset unexpectedly reaches >20,000 names and search exceeds 200ms, add debouncing (150ms delay) as a first response. Server-side search is a last resort. |
| `Intl.Collator("pl")` not supported in a browser | All modern browsers (Chrome 24+, Firefox 29+, Safari 10+, Edge 12+) support `Intl.Collator` with Polish locale. For unsupported browsers, the collator falls back to default locale comparison — search still works but diacritics matching degrades. No mitigation beyond browser support tables. |
| Race condition: `reconsolidate` runs while `data`/`keys` are being updated | JavaScript is single-threaded. The `onValue` callback updates `data` and `keys` synchronously before calling `reconsolidate`. No race condition is possible. |
| New name arrives but search input matches old data | The data-change-triggered `reconsolidate` call (ADR-02-01 §3) re-runs the search with current input, incorporating the new name immediately. |

## Consequences

- The search algorithm is unchanged — zero risk of regression in search correctness.
- The key index becomes dynamic — a trivial code change (reassign `keys = Object.keys(data)` in the listener).
- Newly arrived names are immediately searchable in active sessions.
- No new dependencies, no new components, no build tooling.
- Related: [ADR-02-01](adr-02-01-client-architecture.md) (client architecture handles state mutation and dual-trigger rendering).
