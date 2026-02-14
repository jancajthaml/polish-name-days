# Constraints — Polish Name Days (Realtime)

## Constraints table

| Dimension | Value | Source | Rationale |
|-----------|-------|--------|-----------|
| **Network model** | Public internet only. Clients are browsers on arbitrary networks. No private network, no VPN, no air-gap. | Codebase: GitHub Pages deployment, public static site. Prompt: no network restrictions stated. | The current system serves anonymous public users over HTTPS. The target must do the same. |
| **Operator profile** | Solo developer or very small team. No dedicated ops role. No evidence of infrastructure management capability. | Codebase: zero CI/CD, zero container configs, zero orchestration manifests, zero monitoring, zero dependency management. Single HTML file. | The codebase was deliberately kept operationally trivial. Any solution requiring sustained infrastructure management is suspect. |
| **Hardware profile** | No owned or rented server infrastructure. Current cost is zero (GitHub Pages free tier). | Codebase: pure static hosting, no server component. | Introducing server infrastructure is a new cost and operational burden that did not previously exist. |
| **Component budget** | Maximum 2-3 new infrastructure components beyond the frontend. Current count: 1 (static site). | Derived: operator profile (solo/small team), hardware profile (zero-to-low cost), codebase complexity (170 lines). | A 170-line application does not justify a 10-component infrastructure stack. Every new component must be operated, monitored, and maintained by a team that currently operates nothing. |
| **Internet access** | Always connected. No offline requirement stated. | Prompt: "update realtime" implies connected clients. Codebase: no service worker, no offline cache, no local storage. | Realtime updates are inherently an online feature. Offline degradation should be graceful but is not a primary requirement. |
| **Data volume** | ~4700 unique names, ~6602 name-date pairs. Growth rate: very low (Polish name days are cultural/calendar data, not user-generated content). | Codebase: `name-days.csv` analysis. | This is a small, slowly-growing dataset. Solutions designed for high-volume, high-velocity data are disproportionate. |
| **Data freshness** | Realtime — new names must appear to connected clients without page reload. | Prompt: "if there are new polish names the name days will update realtime." | This is the core requirement. "Realtime" means seconds-to-low-minutes, not hours or days. |
| **Cost sensitivity** | High. Current operational cost is zero. No budget stated. | Codebase: GitHub Pages free tier, no paid services. | Solutions with significant recurring costs (dedicated servers, paid realtime services above free tiers) must be justified against the value of a name-day lookup tool. |
| **Compliance** | None stated. No PII, no financial data, no health data. Name-day data is public cultural information. | Codebase: no auth, no user data, no privacy controls. Prompt: no compliance mentioned. | No regulatory constraints apply. |
| **Client diversity** | Standard modern browsers. Mobile and desktop. | Codebase: `<meta name="viewport">` present, minimal responsive CSS. | Solution must work in all modern browsers. No native app requirement. |

## Invariants

1. **Polish diacritics-insensitive fuzzy search must work correctly for all connected clients at all times.** This is the core user-facing capability — if search breaks, the application is useless.
2. **New name-day data must propagate to connected clients without requiring page reload or redeployment.** This is the stated architectural intent — if data only updates on deploy, the requirement is not met.
3. **The system must remain operatable by a solo developer or very small team without dedicated infrastructure expertise.** The codebase evidence (zero ops tooling, zero dependencies, single HTML file) establishes the operator profile — violating this makes the system unoperatable.
4. **The total number of infrastructure components must not exceed what the operator profile can manage.** Every component needs monitoring, updating, debugging, and incident response. The operator budget is thin.
5. **The existing search UX contract must be preserved: user types → top fuzzy matches with dates appear, updated per keystroke.** The user asked for dynamic data, not a different interface.

## Failure scenarios

### FS-1: Realtime data source becomes unavailable

**What fails**: The component delivering realtime updates (whatever it is) goes down or becomes unreachable.
**Blast radius**: New names stop propagating. If the initial data load also depends on this source, the application may fail entirely on page load — no data, no search, blank page.
**Expected recovery**: The application must still function with the last-known dataset. Initial load must not hard-depend on a realtime connection that may be temporarily down.

### FS-2: Data source corrupted or malformed data pushed

**What fails**: A malformed entry (missing name, invalid date format, duplicate contradicting an existing entry) is pushed to the data source.
**Blast radius**: Client-side parsing fails or produces incorrect results. If the search index is rebuilt on each update, a single bad record could break the entire dataset.
**Expected recovery**: Client-side validation must reject or skip malformed entries. The search index must not be destroyed by a single bad record.

### FS-3: Realtime connection drops mid-session

**What fails**: A client's realtime connection (WebSocket, SSE, or equivalent) is interrupted by network change, sleep/wake, or transient failure.
**Blast radius**: That client stops receiving updates. If the client has no reconnection logic, it silently falls behind.
**Expected recovery**: Automatic reconnection with state reconciliation. The client must know what it missed and catch up, or re-fetch the full dataset.

### FS-4: Operator loses access to the realtime infrastructure provider

**What fails**: The third-party service hosting the realtime infrastructure (if used) changes pricing, discontinues free tier, or experiences prolonged outage.
**Blast radius**: Entire realtime capability lost. If the frontend hard-depends on this service, the application is dead.
**Expected recovery**: The architecture must have an escape hatch — the ability to switch realtime providers or fall back to periodic polling without rewriting the entire frontend.

### FS-5: Dataset grows beyond client-side search performance threshold

**What fails**: The Levenshtein scan is O(n) across all names on every keystroke. If the dataset grows from ~4700 to tens of thousands of names, search becomes sluggish.
**Blast radius**: UI becomes unresponsive. Users abandon the tool.
**Expected recovery**: Current dataset is small and growth rate is very low (cultural data). This is a theoretical concern, not an immediate one. Mitigation: debounce, or move search to server-side if dataset crosses a threshold. But do not pre-optimize for a scale that may never arrive.

### FS-6: Concurrent data writes produce conflicting state

**What fails**: Two operators add name-day data simultaneously. One write overwrites the other, or both produce a merged state that is inconsistent.
**Blast radius**: Data loss or duplicate/contradictory entries.
**Expected recovery**: For a very-low-write-frequency dataset (new Polish name days are added rarely), this is unlikely. A simple last-write-wins or append-only model suffices. Do not introduce distributed consensus for a dataset that changes monthly at most.

### FS-7: Initial page load fails because realtime infrastructure is not yet ready

**What fails**: The realtime data source takes time to initialise (cold start, serverless spin-up). The client connects before data is available.
**Blast radius**: User sees empty results on first load. Poor first impression.
**Expected recovery**: The architecture must support an initial full-dataset load that does not depend on the realtime channel being fully operational. Fallback: serve a static snapshot.

### FS-8: Browser does not support chosen realtime protocol

**What fails**: The client browser (old mobile browser, corporate proxy) blocks WebSocket or SSE.
**Blast radius**: That client gets no realtime updates. If the initial load also requires the realtime protocol, the client gets nothing.
**Expected recovery**: The realtime mechanism must degrade to HTTP polling or the initial load must use plain HTTP fetch. Protocol negotiation or fallback is required.

## Challenged mapping table

Assessment of each prima materia mapping row: is this capability **necessary** in the target environment, **questionable** (uncertain necessity), or an **artifact** of the source environment?

| Capability | Assessment | Rationale |
|-----------|-----------|-----------|
| Data storage | Necessary | Static CSV cannot support realtime updates. A dynamic data store is required — but it must be minimal (managed service or simple append-only store), not a full database server. |
| Data delivery | Necessary | One-time fetch cannot deliver updates after page load. A realtime delivery mechanism is the core of the architectural intent. |
| Data format | Questionable | CSV may remain viable if the data source is still a file (e.g., CSV in a realtime-enabled store). Changing to JSON is likely but not mandatory — depends on the data delivery mechanism chosen. Do not assume JSON without an ADR. |
| Fuzzy search algorithm | Necessary | Must be preserved. Client-side Levenshtein with Polish collator is proven correct and performs adequately at current scale (~4700 names). Moving to server-side adds a component, latency, and operational burden for no demonstrated need. Keep client-side unless proven inadequate. |
| Search result set | Necessary | Must become dynamic. The `keys` array must update when new data arrives. This is a code change, not a new component. |
| UI rendering | Questionable | Vanilla DOM manipulation is functional. Introducing a reactive framework (React, Vue, etc.) adds a build system, dependencies, and complexity. The only rendering gap is handling data-change events alongside keystrokes — this can be solved with a few lines of vanilla JS (re-extract keys, re-run search if input is non-empty). A framework is not necessary unless other requirements demand it. |
| Hosting | Necessary | GitHub Pages cannot serve realtime connections (no server-side capability). Hosting must change or be augmented — but consider a hybrid: keep GitHub Pages for static assets, add a minimal realtime service alongside. |
| Polish locale handling | Necessary | Must be preserved. `Intl.Collator` is a browser standard — no change needed regardless of architecture. |
| Build system | Questionable | A build system is only needed if a framework or module bundler is introduced. If the solution stays vanilla JS with a realtime client library loaded via CDN, zero-build can be preserved. Do not introduce a build system without a demonstrated need. |
| Test infrastructure | Questionable | Tests are valuable but the current system has none. Introducing a test framework adds tooling the operator must maintain. Proportional response: add tests if the codebase complexity increases, but do not make a test framework a blocking requirement for a ~200-line application. |
