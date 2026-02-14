# Prima Materia — Polish Name Days

The current system is a static single-page application that lets users search Polish name days by name using fuzzy matching. It consists of a single HTML file with inline JavaScript and a CSV data file (~6602 name-date pairs, ~4700 unique names), hosted on GitHub Pages with no backend. The target is a dynamic system where name-day data updates in realtime when new Polish names are added, without requiring redeployment or page reload.

## Domain capabilities

| Domain | Purpose | External couplings |
|--------|---------|-------------------|
| Name-day lookup | Search Polish name days by entering a name and receiving the closest matches with their associated dates | GitHub Pages (hardcoded CSV fetch URL) |
| Fuzzy search | Polish-diacritics-aware approximate string matching using Levenshtein distance, returning top 3 results within distance threshold | Browser `Intl.Collator` API (Polish locale) |
| Data ingestion | Load and parse name-day data from CSV into an in-memory name→dates mapping on page load | GitHub Pages (static CSV file hosting) |
| UI rendering | Display matched names and their dates, rebuilding the result set on every keystroke | None |

## Current → desired mapping

| Capability | Current | Desired | ADR | Challenged |
|-----------|---------|---------|-----|-----------|
| Data storage | Static CSV file (`name-days.csv`, 6602 rows) committed to git repo | Firebase Realtime Database — JSON document store with realtime sync | [ADR-01-01](adr/01-data/adr-01-01-realtime-data-service.md) | Necessary — static CSV cannot support realtime updates |
| Data delivery | One-time HTTP GET fetch from hardcoded GitHub Pages URL on page load | Firebase `onValue` listener — initial snapshot + incremental push updates | [ADR-01-01](adr/01-data/adr-01-01-realtime-data-service.md) | Necessary — one-time fetch cannot deliver post-load updates |
| Data format | CSV (`Name,DD.MM`) parsed client-side via `String.split` | JSON name-to-dates map in Firebase RTDB; client transforms snapshot to `{ name: [dates] }` | [ADR-01-02](adr/01-data/adr-01-02-data-schema.md) | Questionable → resolved: JSON follows naturally from Firebase RTDB choice |
| Fuzzy search algorithm | Custom client-side Levenshtein distance with `Intl.Collator("pl")` | Preserved unchanged — client-side Levenshtein with Polish collator, dynamic key index | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) | Necessary — kept client-side as assessed |
| Search result set | Precomputed `Object.keys(data)` array, static after load | Mutable `keys` array re-extracted on every `onValue` callback | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) | Necessary — code change, not new component |
| UI rendering | Vanilla DOM manipulation — full clear + rebuild on each keyup event | Vanilla JS preserved — dual-trigger rendering (keystroke + data change) | [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) | Questionable → resolved: vanilla JS with minimal additions, no framework |
| Hosting | GitHub Pages (static file hosting, no server-side capability) | Hybrid — GitHub Pages for static assets + Firebase RTDB for realtime data | [ADR-03-01](adr/03-infrastructure/adr-03-01-hosting-strategy.md) | Necessary — hybrid approach keeps GitHub Pages, adds Firebase |
| Polish locale handling | Browser `Intl.Collator("pl", { sensitivity: "base" })` | Preserved unchanged — browser standard | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) | Necessary — no change needed |
| Build system | None — raw HTML/JS/CSS served directly | Preserved zero-build — Firebase compat SDK loaded via CDN `<script>` tag | [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) | Questionable → resolved: zero-build preserved via CDN loading |
| Test infrastructure | None | Not addressed — proportional to codebase size | — | Questionable — deferred; not blocking for ~200-line app |
