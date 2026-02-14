# Phase 01 — Raw Findings

## 1. Directory survey

```
/workspace/
├── index.html          # Single-page application (HTML + inline CSS + inline JS)
├── name-days.csv       # Polish name-day data (6603 lines, ~4700 unique names)
└── README.md           # Minimal — link to GitHub Pages deployment
```

- **Language**: Vanilla JavaScript (ES6+), HTML5, CSS3 — all inline in `index.html`.
- **Framework**: None. No libraries, no build tools, no dependency manager.
- **Build system**: None. No `package.json`, no bundler, no transpiler.
- **Infrastructure artifacts**: None. No CI/CD, no Dockerfile, no orchestration manifests. Deployed as a static site via GitHub Pages.

## 2. Dependency scan

No dependency files exist. Zero external runtime or development dependencies. The application is entirely self-contained in a single HTML file and a CSV data file.

## 3. External coupling

| Coupling | Type | Where | Evidence |
|----------|------|-------|----------|
| GitHub Pages hosting | Platform | `index.html:95` | `fetch('https://jancajthaml.github.io/polish-name-days/name-days.csv')` — hardcoded absolute URL to self-hosted CSV on GitHub Pages |
| GitHub Pages deployment | Platform | `README.md:3` | Live URL: `https://jancajthaml.github.io/polish-name-days/` |
| `Intl.Collator` (browser API) | Runtime | `index.html:45` | Polish locale collator `Intl.Collator("pl", { sensitivity: "base" })` — depends on browser i18n support |

No vendor SDKs, no SaaS integrations, no database drivers, no cloud-specific dependencies.

## 4. Communication patterns

| Pattern | Protocol | Where used | Vendor / integration dependency |
|---------|----------|-----------|-------------------------------|
| Static CSV fetch | HTTP GET | `index.html:95` — `fetch('https://jancajthaml.github.io/polish-name-days/name-days.csv')` | GitHub Pages (hardcoded URL) |

- Communication is **unidirectional**: client fetches CSV once on page load.
- No WebSocket, no background jobs, no message queues, no pub/sub, no RPC.
- No server-side component — purely client-side.

## 5. Storage patterns

| Storage | Type | Where | Details |
|---------|------|-------|---------|
| `name-days.csv` | Static flat file (CSV) | Root directory | Format: `Name,DD.MM` — 6602 data rows + 1 header. ~4700 unique names, many with multiple dates. |
| In-memory JavaScript object | Client-side runtime | `index.html:107-119` | CSV parsed into `{ name: [date, date, ...] }` object, held in closure. |

- No database, no object storage, no caching layer, no session store.
- Data is **immutable at runtime** — loaded once from static file, never updated.

## 6. Authentication & authorization

None. Public static site with no authentication, no session management, no RBAC, no tokens.

## 7. Configuration & secrets

None. No environment variables, no config files, no secret managers. The only configurable value is the hardcoded fetch URL in `index.html:95`.

## 8. Infrastructure

| Component | Description |
|-----------|-------------|
| GitHub Pages | Static file hosting. No custom domain configuration detected. Uses `<base href="/" />` in `index.html:6`. |

No CI/CD pipelines, no container definitions, no orchestration manifests, no `CNAME` file.

## 9. Multi-tenancy signals

None. Single-user, single-tenant, stateless static page.

## 10. Language patterns (Pattern Translator)

| Pattern | Type | Where used | Frequency | Structural |
|---------|------|-----------|-----------|-----------|
| IIFE (Immediately Invoked Function Expression) | Scope encapsulation | `index.html:44` (Levenshtein), `index.html:138` (main app) | 2 instances | Yes — encapsulates all application logic |
| Global object attachment | Module pattern | `index.html:50` — `window.Levenshtein` | 1 instance | Yes — exposes Levenshtein as global API |
| Closure-based data encapsulation | Data hiding | `index.html:139-165` — `data` and `keys` captured in closure | 1 instance | Yes — entire app state lives in closure |
| Promise chaining (`fetch().then().then()`) | Async data loading | `index.html:94-120` | 1 chain (4 `.then()` calls) | Yes — data loading pipeline |
| DOM manipulation via `document.createElement` / `innerHTML` | UI rendering | `index.html:122-136`, `index.html:143-161` | Pervasive | Yes — only rendering approach |
| Event listener (`keyup`) | User input handling | `index.html:164` | 1 instance | Yes — sole interaction mechanism |
| `Intl.Collator` for locale-aware comparison | Polish diacritics handling | `index.html:45` | 1 instance | Yes — core to fuzzy search correctness |
| Custom Levenshtein distance implementation | Fuzzy string matching | `index.html:51-91` | 1 instance | Yes — entire search algorithm |
| CSV parsing via `String.split` | Data ingestion | `index.html:100-119` | 1 instance | Yes — only data format handler |
| Full result-set clearing + rebuild on each keyup | Render strategy | `index.html:143` — `innerHTML = ""` then rebuild | Every keystroke | Yes — naive re-render pattern |

## 11. External contracts

| Contract | Type | Endpoint / schema | Where defined |
|----------|------|------------------|--------------|
| CSV data format | Data contract | `Name,DD.MM` — comma-separated, header row `Name,DD.MM`, data rows with Polish names and day.month dates | `name-days.csv:1` (header), consumed at `index.html:109` |
| GitHub Pages URL | HTTP endpoint | `https://jancajthaml.github.io/polish-name-days/name-days.csv` | `index.html:95` |

No REST API, no GraphQL, no WebSocket events, no webhook payloads, no CLI interface.

## 12. Test architecture

None. No test files, no test framework, no test configuration, no test directories.

## 13. Module dependencies

Single file (`index.html`). No module system, no imports, no `require()` statements. All code is inline in one `<script>` block.

Internal dependency graph:
```
load() → fetch CSV → parse → build name→dates map
       ↓
reconsolidate() → Levenshtein.get() → DOM update
       ↑
keyup event listener
```

---

## Domain capabilities table

| Domain | Key files / classes | Purpose | External couplings |
|--------|-------------------|---------|-------------------|
| Name-day lookup | `index.html` (entire app), `name-days.csv` | Search Polish name days by name with fuzzy matching | GitHub Pages (CSV hosting, hardcoded URL) |
| Fuzzy search | `index.html:44-92` — `window.Levenshtein` | Locale-aware Levenshtein distance for Polish name matching | Browser `Intl.Collator` API (Polish locale) |
| Data ingestion | `index.html:94-120` — `load()` | Fetch and parse static CSV into in-memory name→dates mapping | GitHub Pages (CSV fetch URL) |
| UI rendering | `index.html:122-165` | Display top 3 closest name matches with their dates | None (vanilla DOM manipulation) |

## Communication patterns table

| Pattern | Protocol | Where used | Vendor / integration dependency |
|---------|----------|-----------|-------------------------------|
| One-time static file fetch | HTTP GET | `index.html:95` — page load triggers CSV download | GitHub Pages (absolute URL hardcoded) |
| DOM event-driven UI update | Browser event (`keyup`) | `index.html:164` — each keystroke triggers full search + re-render | None |

## Terse raw notes

- The entire application is ~170 lines in a single HTML file. No build, no dependencies, no backend.
- The Levenshtein implementation is custom, uses `Intl.Collator("pl")` for Polish diacritics-insensitive matching. It mutates shared arrays (`prevRow`, `str2Char`) for performance — not thread-safe but irrelevant in single-threaded browser context.
- The CSV fetch uses an **absolute hardcoded URL** to GitHub Pages, meaning the data source is coupled to the deployment location. The CSV file also exists locally in the repo.
- Search results are limited to top 3 matches with Levenshtein distance < 5.
- The `name-days.csv` contains ~6602 name-date pairs across ~4700 unique names. Data is static — no mechanism exists to update it at runtime or add new names dynamically.
- The `{ useCollator: true }` option passed to `Levenshtein.get()` at `index.html:146` is **dead code** — the option is not read inside the function; the collator is always used.
- The `<base href="/" />` tag at `index.html:6` may cause path resolution issues depending on deployment configuration.
- **Architectural intent** (from user): make the user experience dynamic so if there are new Polish names, the name days will update in realtime. Currently, the data is static — loaded once from a CSV, no update mechanism, no push capability, no polling, no realtime infrastructure.
