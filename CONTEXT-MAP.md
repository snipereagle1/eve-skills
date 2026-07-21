# Context Map

This repo is a monorepo of EVE Online agent skills. Each skill is a bounded context with its own glossary. This map names the contexts and the language they share.

## Contexts

- [ESI](./skills/esi/CONTEXT.md) — discipline for consuming EVE Online's ESI HTTP API correctly: auth, error budget, caching, pagination, versioning.
- [SDE](./skills/sde/CONTEXT.md) — legibility for the Static Data Export, the offline dataset of static game data: the domain map (datasets + join keys), the build-number sync model, and the JSONL format encodings.

## Relationships

- **ESI ↔ SDE — the datasource boundary.** The two contexts share one load-bearing term: which datasource answers a given question. Static/reference data (type names, topology, dogma) belongs to the SDE; dynamic/character/live/market data belongs to ESI. The ESI context defines the **SDE boundary** from its side; the SDE context states only its own half (the SDE is static-only — no prices, no live state) and **points forward** to a planned `sde-vs-esi` skill that will own the full trade-off analysis. The shared definition must stay consistent across both — a change to the boundary is a change to both glossaries. Neither skill adjudicates "which to use" today; that is deliberately deferred.
