# Frontend

| # | ADR | Summary |
|---|-----|---------|
| 02-01 | [Client Architecture — Vanilla JS with Firebase SDK](adr-02-01-client-architecture.md) | Retain vanilla JavaScript, add Firebase compat SDK via CDN, mutable state with `onValue` listener, static fallback, zero-build |
| 02-02 | [Search Preservation — Client-side Levenshtein with Dynamic Index](adr-02-02-search-preservation.md) | Preserve existing Levenshtein algorithm unchanged, make key index dynamic, re-run search on data change |
