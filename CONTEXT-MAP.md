# Context Map

This repo is a monorepo of EVE Online agent skills. Each skill is a bounded context with its own glossary. This map names the contexts and the language they share.

## Contexts

- [ESI](./skills/esi/CONTEXT.md) — discipline for consuming EVE Online's ESI HTTP API correctly: auth, error budget, caching, pagination, versioning.
- **SDE** *(planned)* — discipline for the Static Data Export, the offline dataset of static game data. No `CONTEXT.md` yet; created when the skill is built.

## Relationships

- **ESI ↔ SDE — the datasource boundary.** The two contexts share one load-bearing decision: which datasource answers a given question. Static/reference data (type names, topology, dogma) belongs to the SDE; dynamic/character/live/market data belongs to ESI. The ESI context defines the **SDE boundary** term from its side; the SDE context will define its half when it exists. The definition must stay consistent across both — a change to the boundary is a change to both glossaries.
