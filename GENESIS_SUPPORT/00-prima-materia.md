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
| Data storage | Static CSV file (`name-days.csv`, 6602 rows) committed to git repo | TBD — dynamic data store that supports additions without redeployment | TBD | |
| Data delivery | One-time HTTP GET fetch from hardcoded GitHub Pages URL on page load | TBD — realtime delivery mechanism that pushes or streams updates to connected clients | TBD | |
| Data format | CSV (`Name,DD.MM`) parsed client-side via `String.split` | TBD — format compatible with realtime delivery | TBD | |
| Fuzzy search algorithm | Custom client-side Levenshtein distance with `Intl.Collator("pl")` | TBD — must remain Polish-diacritics-aware; may stay client-side or move to backend | TBD | |
| Search result set | Precomputed `Object.keys(data)` array, static after load | TBD — must update when new names arrive at runtime | TBD | |
| UI rendering | Vanilla DOM manipulation — full clear + rebuild on each keyup event | TBD — must reflect data changes without page reload | TBD | |
| Hosting | GitHub Pages (static file hosting, no server-side capability) | TBD — must support realtime data delivery infrastructure | TBD | |
| Polish locale handling | Browser `Intl.Collator("pl", { sensitivity: "base" })` | TBD — must preserve Polish diacritics-insensitive matching | TBD | |
| Build system | None — raw HTML/JS/CSS served directly | TBD — may remain zero-build or introduce tooling if framework chosen | TBD | |
| Test infrastructure | None | TBD | TBD | |
