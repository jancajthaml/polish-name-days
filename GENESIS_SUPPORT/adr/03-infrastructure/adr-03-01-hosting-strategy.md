# ADR-03-01: Hosting Strategy — Hybrid GitHub Pages + Firebase

## Context

The current system is hosted entirely on GitHub Pages (free static file hosting). The topology identifies two server-side deployment units: a Static Asset Host (existing) and a Realtime Data Service (new). PRD-005 requires ≤3 infrastructure components, free-tier cost, and no specialist expertise. The Constraint Synthesis assessed hosting as "necessary — GitHub Pages cannot serve realtime; consider hybrid."

See also: [PRD-004](../../prd/prd-004-system-resilience.md), [PRD-005](../../prd/prd-005-operational-simplicity.md)

## Decision

We choose a **hybrid hosting strategy**:

- **GitHub Pages** continues to serve static assets (HTML, CSS, JS, static fallback JSON). No change to the existing deployment mechanism (push to `main` branch or `gh-pages` branch → automatic deployment).
- **Firebase Realtime Database** (chosen in [ADR-01-01](../01-data/adr-01-01-realtime-data-service.md)) serves as the Realtime Data Service. It is not a hosting platform — it is a data service that browsers connect to directly.

### Why not migrate static hosting to Firebase Hosting?

Firebase Hosting exists and could serve static files. However:
- GitHub Pages already works and costs nothing.
- Migrating adds effort and risk for zero functional benefit.
- Keeping static hosting on GitHub Pages means the codebase repository directly deploys the site — simpler mental model for the operator.
- Separating static hosting from the data service reduces blast radius: if Firebase has an outage, the site still loads (static fallback works).

### Deployment flows

| What | How | Where |
|------|-----|-------|
| Static assets (HTML, JS, CSS) | Git push to repository → GitHub Pages auto-deploys | GitHub Pages |
| Static fallback JSON | Git push (bundled with static assets) | GitHub Pages |
| Name-day data (live) | Operator writes via Firebase Console or REST API | Firebase RTDB |
| Firebase security rules | `firebase deploy --only database` or manual via Firebase Console | Firebase |

### CORS and cross-origin

The browser loads static assets from `jancajthaml.github.io` and connects to Firebase at `*.firebaseio.com`. These are different origins. Firebase RTDB handles CORS natively — no custom CORS configuration needed. The Firebase SDK manages the cross-origin connection transparently.

### Alternatives considered

| Alternative | Why rejected |
|------------|-------------|
| Migrate everything to Firebase Hosting | Adds migration effort, changes deployment workflow, creates single-provider dependency. No functional benefit over GitHub Pages for static files. |
| Self-hosted static server (Nginx, Caddy) | Violates operator profile and cost constraints. GitHub Pages is free and zero-maintenance. |
| Netlify / Vercel for static hosting | Functionally equivalent to GitHub Pages for this use case. Switching adds migration effort for no gain. GitHub Pages is already integrated with the repository. |

## Failure modes and mitigations

| Failure mode | Mitigation |
|-------------|-----------|
| GitHub Pages outage | New page loads fail. Existing sessions continue (assets already loaded, Firebase connection independent). This is the same failure mode as the current system — no regression. |
| Firebase outage while GitHub Pages is up | Site loads successfully. Firebase listener times out after 5 seconds (ADR-02-01 §4). Client falls back to static JSON snapshot served from GitHub Pages. Search works on static data. Realtime updates resume when Firebase recovers. |
| Both GitHub Pages and Firebase are down simultaneously | New page loads fail entirely. Existing sessions may continue on cached assets + in-memory data. Extremely unlikely (two independent providers). No mitigation beyond what is already in place. |
| `<base href="/" />` tag causes path resolution issues | The current `index.html` has `<base href="/" />` which may interfere with relative paths. This should be reviewed and potentially removed or adjusted during implementation to ensure the static fallback JSON and Firebase SDK load correctly. |

## Consequences

- The operator manages two services: GitHub repository (for code) and Firebase project (for data). Both have free tiers. Both have web consoles.
- Static assets and live data have independent availability — partial failures are recoverable.
- The existing GitHub Pages deployment workflow is unchanged.
- A new one-time setup step is added: create Firebase project, configure security rules, import initial data.
- Related: [ADR-01-01](../01-data/adr-01-01-realtime-data-service.md) (Firebase RTDB), [ADR-02-01](../02-frontend/adr-02-01-client-architecture.md) (client fallback logic).
