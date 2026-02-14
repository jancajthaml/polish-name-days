# Solution Design — Polish Name Days (Realtime)

## Scope

This design covers the architectural transformation of the Polish Name Days application from a static single-page site (one-time CSV fetch, no update mechanism) to a dynamic system where name-day data updates in realtime when new Polish names are added — without redeployment or page reload. The design preserves the core user experience: fuzzy Polish-name search with diacritics-insensitive matching.

## Key decisions

- **Firebase Realtime Database** as the single new infrastructure component — absorbs data storage, realtime push delivery, and data management into one managed service on the free tier ([ADR-01-01](../GENESIS_SUPPORT/adr/01-data/adr-01-01-realtime-data-service.md))
- **JSON name-to-dates map** schema in Firebase RTDB with server-side validation rules ([ADR-01-02](../GENESIS_SUPPORT/adr/01-data/adr-01-02-data-schema.md))
- **Vanilla JavaScript preserved** — no framework, no build system; Firebase compat SDK loaded via CDN `<script>` tag; static JSON fallback for resilience ([ADR-02-01](../GENESIS_SUPPORT/adr/02-frontend/adr-02-01-client-architecture.md))
- **Levenshtein search algorithm unchanged** — dynamic key index re-extracted on data changes; dual-trigger rendering for immediate visibility of new names ([ADR-02-02](../GENESIS_SUPPORT/adr/02-frontend/adr-02-02-search-preservation.md))
- **Hybrid hosting** — GitHub Pages for static assets (unchanged), Firebase RTDB for realtime data; independent availability ([ADR-03-01](../GENESIS_SUPPORT/adr/03-infrastructure/adr-03-01-hosting-strategy.md))

## How to read this doc set

| Document | Location | Purpose |
|----------|----------|---------|
| Architecture | [GENESIS_RESULT/architecture/](architecture/) | High-level architecture diagram and narrative |
| Transformation plan | [GENESIS_RESULT/plan/transformation-plan.md](plan/transformation-plan.md) | How to get from current state to target state |
| Work packages | [GENESIS_RESULT/plan/work-packages.md](plan/work-packages.md) | Ordered, executable work units with acceptance criteria |
| Validation matrix | [GENESIS_RESULT/plan/validation-matrix.md](plan/validation-matrix.md) | How to verify each step and the overall transformation |
| Prima materia | [GENESIS_SUPPORT/00-prima-materia.md](../GENESIS_SUPPORT/00-prima-materia.md) | Domain distillation and mapping |
| Constraints | [GENESIS_SUPPORT/01-constraints.md](../GENESIS_SUPPORT/01-constraints.md) | Target environment constraints, invariants, failure scenarios |
| Topology | [GENESIS_SUPPORT/02-topology.md](../GENESIS_SUPPORT/02-topology.md) | System topology, deployment units, component minimisation |
| Pattern map | [GENESIS_SUPPORT/03-pattern-map.md](../GENESIS_SUPPORT/03-pattern-map.md) | Pattern inventory, mappings, unmappable patterns, contracts |
| PRDs | [GENESIS_SUPPORT/prd/](../GENESIS_SUPPORT/prd/) | Product Requirements Documents |
| ADRs | [GENESIS_SUPPORT/adr/](../GENESIS_SUPPORT/adr/) | Architecture Decision Records |
