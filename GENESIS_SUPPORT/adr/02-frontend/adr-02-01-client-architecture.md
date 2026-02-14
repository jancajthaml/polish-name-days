# ADR-02-01: Client Architecture — Vanilla JS with Firebase SDK

## Context

The current application is a single HTML file with inline vanilla JavaScript (~170 lines). It uses IIFEs for scope, a global `Levenshtein` object, closure-based state, and promise-chained CSV fetching. The topology keeps all search and rendering logic in the Browser Client, with no framework component. PRD-003 requires the search to incorporate dynamically arriving names. PRD-004 requires graceful degradation on data source failure. PRD-005 requires operational simplicity — no build system, no framework runtime to maintain.

The Pattern Map (03-pattern-map.md) identifies three high-risk patterns: closure-based immutable state (#3), one-time fetch chain (#4), and static key extraction (#5). These must be refactored to support mutable, realtime data.

See also: [PRD-003](../../prd/prd-003-search-with-dynamic-data.md), [PRD-004](../../prd/prd-004-system-resilience.md), [PRD-005](../../prd/prd-005-operational-simplicity.md)

## Decision

We choose to **retain vanilla JavaScript with no framework**, adding only the Firebase RTDB SDK loaded via CDN `<script>` tag. The client architecture changes are:

### 1. Replace one-time fetch with Firebase `onValue` listener

The `load()` function and its promise chain are replaced by a single Firebase `onValue` listener on `/nameDays`. This listener:
- Fires immediately with the initial full snapshot (replacing the HTTP GET fetch).
- Fires again on every subsequent change (providing realtime updates).
- Handles reconnection and re-synchronisation automatically (Firebase SDK built-in).

### 2. Make state mutable

The `data` object and `keys` array move from closure-captured constants to module-level mutable variables. On every `onValue` callback:
- `data` is rebuilt from the Firebase snapshot (JSON → `{ name: [dates] }` map).
- `keys` is re-extracted via `Object.keys(data)`.
- If the user has typed something in the search input, the search is re-run to reflect the new data.

### 3. Dual-trigger rendering

The `reconsolidate` function (search + render) is now invoked by two triggers:
- **User keystroke** (`keyup` event) — same as current.
- **Data change** (`onValue` callback) — new. If the search input is non-empty, re-run `reconsolidate` with the current input value so newly arrived names appear immediately.

### 4. Static fallback for resilience

A static JSON snapshot of the current dataset is embedded in the HTML file (or served as a separate `.json` file from GitHub Pages). If the Firebase connection fails on initial load (timeout after 5 seconds), the client falls back to this snapshot. Search works on the static data. Realtime updates resume when the connection recovers.

### 5. Zero-build preservation

The Firebase SDK is loaded via:
```html
<script src="https://www.gstatic.com/firebasejs/10.x/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.x/firebase-database-compat.js"></script>
```
No `npm`, no bundler, no transpiler. The `compat` (compatibility) SDK variant supports the `<script>` tag pattern without ES module build tooling.

### 6. Client-side data validation

Every entry received from Firebase is validated before merging into the in-memory dataset:
- Name must be a non-empty string.
- Each date must match `/^[0-3]\d\.[0-1]\d$/`.
- Entries failing validation are silently skipped (logged to console for debugging).

### Alternatives considered

| Alternative | Why rejected |
|------------|-------------|
| React / Vue / Svelte framework | Adds a build system (Webpack/Vite), dependency management (npm), and a framework runtime. Violates PRD-005 (1-hour cold-start deploy, no specialist expertise). The only rendering gap (data-change re-render) is solvable with 3 lines of vanilla JS. |
| Firebase modular SDK (tree-shakeable ES modules) | Requires a bundler (Webpack/Rollup) to tree-shake unused modules. Not compatible with zero-build `<script>` tag loading. The compat SDK is adequate for this application's needs. |
| Web Components | More boilerplate than vanilla DOM manipulation for this simple UI. No benefit for a single-page, single-component application. |
| Lit / Alpine.js (lightweight frameworks) | Smaller than React but still add a dependency and a mental model. The current vanilla approach is adequate. |

## Failure modes and mitigations

| Failure mode | Mitigation |
|-------------|-----------|
| Firebase SDK fails to load (CDN unreachable, ad blocker) | The `<script>` tag includes an `onerror` handler. On failure, the client loads the static JSON fallback and disables realtime features. Search works on static data. |
| Firebase `onValue` never fires (connection timeout) | A 5-second timeout starts on page load. If `onValue` has not fired, the client loads the static fallback. If `onValue` fires later, the static data is replaced with the live data. |
| `onValue` fires with `null` (empty database) | Client checks for null/empty snapshot. If empty, falls back to static data and logs a warning. |
| Rapid successive `onValue` callbacks cause rendering flicker | The `reconsolidate` function is idempotent (clears and rebuilds). Rapid calls may cause brief flicker but no incorrect state. If needed, debounce the data-change-triggered re-render at 100ms. |
| Memory leak from listener not being detached | The listener is attached once on page load and never detached (single-page app with page-lifetime scope). No leak — the listener lifetime matches the page lifetime. |
| Stale closure references after state mutation | State variables (`data`, `keys`) are module-level, not captured in nested closures. The `reconsolidate` function always reads the current values. No stale closures. |

## Consequences

- The application gains a runtime dependency on the Firebase RTDB compat SDK (~40 KB gzipped, loaded from Google CDN).
- The code structure changes from IIFE + closure to module-level mutable state with a Firebase listener.
- The zero-build approach is preserved — no `package.json`, no bundler, no transpiler.
- A static JSON fallback ensures the application never shows a blank page, even if Firebase is unreachable.
- Related: [ADR-01-01](../01-data/adr-01-01-realtime-data-service.md) (Firebase RTDB choice), [ADR-01-02](../01-data/adr-01-02-data-schema.md) (data schema).
