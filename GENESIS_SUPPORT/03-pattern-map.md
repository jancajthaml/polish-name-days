# Pattern Map — Polish Name Days

The current application is vanilla JavaScript (ES6+) with no framework. The architectural intent — making name-day data update in realtime — does not involve a language change but will likely require re-expressing several structural patterns: the one-time data load must become a persistent subscription, the static in-memory dataset must become mutable, and the rendering approach must handle data arriving after initial load. The source and target are both JavaScript/HTML, but the idioms will shift from static-fetch-and-render to reactive-data-and-update.

## Pattern inventory

| # | Source pattern | Idiom / example | Frequency | Structural | Risk |
|---|---------------|----------------|-----------|-----------|------|
| 1 | IIFE scope encapsulation | `(function() { ... }())` wrapping Levenshtein and main app | 2 instances | Yes | Low — mechanical refactor if modules introduced |
| 2 | Global object attachment | `window.Levenshtein = { get: function(...) {...} }` | 1 instance | Yes | Low — replaceable with module export |
| 3 | Closure-based state | `load().then(function(data) { const keys = Object.keys(data); ... })` — entire app state captured in closure, immutable after load | 1 instance | Yes | High — closure assumes data never changes; realtime breaks this assumption |
| 4 | One-time fetch + promise chain | `fetch(url).then().then().then().then()` — four chained transforms, executed once on page load | 1 chain | Yes | High — must be replaced with persistent data subscription |
| 5 | Static key extraction | `Object.keys(data)` computed once after load, used for all subsequent searches | 1 instance | Yes | High — key list must update when new names arrive |
| 6 | Full DOM clear + rebuild | `document.getElementById("result").innerHTML = ""` followed by `createElement` loop on every keyup | Every keystroke | Yes | Medium — inefficient but functional; becomes more problematic with realtime data changes triggering re-renders |
| 7 | `Intl.Collator` for Polish locale | `Intl.Collator("pl", { sensitivity: "base" })` used inside Levenshtein for diacritics-insensitive comparison | 1 instance | Yes | Low — browser API, not pattern-dependent |
| 8 | Custom Levenshtein with shared mutable arrays | `prevRow` and `str2Char` arrays reused across calls for performance; collator always active | 1 instance | Yes | Medium — works in single-threaded browser; if search moves to worker or server, shared state is unsafe |
| 9 | CSV parsing via string split | `data.split(/\r?\n/)` then `line.split(',')` | 1 instance | Yes | Medium — CSV format may change if data source changes |
| 10 | Keyup event listener for search trigger | `document.getElementById("input_name").addEventListener("keyup", reconsolidate)` | 1 instance | Yes | Low — standard DOM event, survives most refactors |

## Pattern mapping

| # | Source pattern | Target equivalent | Semantic gap | Mitigation | ADR |
|---|---------------|------------------|-------------|------------|-----|
| 1 | IIFE scope encapsulation | Retained — vanilla JS, no modules | None — zero-build preserved | No change; scope remains in script | [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) |
| 2 | Global object attachment (`window.Levenshtein`) | Retained — no module system | None — direct preservation | No change | [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) |
| 3 | Closure-based immutable state | Module-level mutable variables | Semantic shift: state is no longer immutable; must handle additions at runtime | Mutable `data` and `keys`; `onValue` callback updates both | [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) |
| 4 | One-time fetch + promise chain | Firebase `onValue` listener | Fundamental shift: one-shot load → persistent connection with incremental updates | Single `onValue` on `/nameDays`; initial snapshot + updates from same channel | [ADR-01-01](adr/01-data/adr-01-01-realtime-data-service.md), [ADR-02-01](adr/02-frontend/adr-02-01-client-architecture.md) |
| 5 | Static key extraction (`Object.keys`) | Dynamic key set re-extracted on `onValue` | Gap: key list must update when new names arrive | Re-extract `keys = Object.keys(data)` in listener callback | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) |
| 7 | `Intl.Collator` Polish locale | Same — `Intl.Collator` is a browser standard | None — direct preservation | No change needed | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) |
| 8 | Custom Levenshtein with shared arrays | Same algorithm, unchanged | Performance gap: linear scan; acceptable at current scale | No change; debounce if dataset grows (ADR-02-02) | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) |
| 10 | Keyup event listener | Same — keyup triggers reconsolidate | None — direct preservation | No change | [ADR-02-02](adr/02-frontend/adr-02-02-search-preservation.md) |

## Unmappable patterns

| # | Source pattern | Frequency | Why unmappable | Proposed workaround | Impact |
|---|---------------|-----------|---------------|--------------------|---------| 
| 6 | Full DOM clear + rebuild on keyup | Every keystroke | Not unmappable per se, but the pattern assumes a static dataset — when combined with realtime data arrivals, clearing and rebuilding the DOM creates a conflict: should a data update mid-typing trigger a re-render? The current pattern has no concept of external data change events. | Introduce a rendering strategy that distinguishes user-triggered re-renders (keystroke) from data-triggered re-renders (new name arrived). May require a reactive framework or explicit render scheduling. | The re-render model changes from "user keystroke → rebuild" to "user keystroke OR data change → rebuild". Without this, new names are invisible until the next keystroke. |
| 9 | CSV parsing via string split | 1 instance | If the data delivery mechanism changes from CSV file to a structured format (JSON from a realtime source), the CSV parser becomes irrelevant. The pattern itself is not unmappable — it simply ceases to exist. | Replace with whatever deserialization the realtime data source provides (e.g., JSON.parse). | Data ingestion pipeline changes entirely; the `load()` function is replaced, not refactored. |

## External contract inventory

| # | Contract | Type | Endpoint / schema | Must preserve |
|---|----------|------|------------------|--------------|
| 1 | CSV data format | Data schema | `Name,DD.MM` — header row, then `{PolishName},{DD}.{MM}` per row | No — the CSV format is an internal implementation detail, not a public API. If the data source changes, the format may change. The semantic contract (name → list of dates) must be preserved. |
| 2 | GitHub Pages URL | HTTP endpoint | `https://jancajthaml.github.io/polish-name-days/name-days.csv` | No — hardcoded to current hosting. Will change if data source or hosting changes. |
| 3 | Search UX contract | User-facing behaviour | User types a name → sees top 3 fuzzy matches with their dates, updated per keystroke | Yes — this is the core user-facing contract. Must be preserved regardless of architectural changes. |
| 4 | Polish diacritics handling | Behavioural contract | Searching "Zofia" matches "Żofia" and vice versa; search is case-insensitive and diacritics-insensitive for Polish characters | Yes — correctness of Polish name matching is a core requirement. |
