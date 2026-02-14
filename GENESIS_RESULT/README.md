# Solution Design — Polish Name Days (Realtime)

## Scope

This design covers the architectural transformation of the Polish Name Days application from a static single-page site (one-time CSV fetch, no update mechanism) to a dynamic system where name-day data updates in realtime when new Polish names are added — without redeployment or page reload. The design preserves the core user experience: fuzzy Polish-name search with diacritics-insensitive matching.

## Key decisions

*To be populated as ADRs are written during Phase 04.*

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
