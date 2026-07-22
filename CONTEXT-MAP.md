# Context Map

This repo is a monorepo of EVE Online agent skills. Each skill is a bounded context with its own glossary. This map names the contexts and the language they share.

## Contexts

- [ESI](./skills/esi/CONTEXT.md) — discipline for consuming EVE Online's ESI HTTP API correctly: auth, error budget, caching, pagination, versioning.
- [SDE](./skills/sde/CONTEXT.md) — legibility for the Static Data Export, the offline dataset of static game data: the domain map (datasets + join keys), the build-number sync model, and the JSONL format encodings.
- [SDE-vs-ESI](./skills/sde-vs-esi/CONTEXT.md) — the datasource decision the other two defer: which source answers a given question (ESI, the SDE, or both), and how to join them. The front door the `esi` and `sde` skills point up to.

## Relationships

- **ESI ↔ SDE — the datasource boundary.** All three contexts share one load-bearing term: which datasource answers a given question. Static/reference data (type names, topology, dogma) belongs to the SDE; dynamic/character/live/market data belongs to ESI. The `esi` and `sde` contexts each state only their own half of the **SDE boundary** (ESI from its side; the SDE as static-only — no prices, no live state) and **defer up** to `sde-vs-esi` for the full call. A change to the boundary is a change to all three glossaries.
- **`esi`, `sde` → `sde-vs-esi` — defer up (front door).** `sde-vs-esi` owns the routing decision and the ESI↔SDE join. The other two are model-invoked on their own API and would otherwise fire before a source is chosen, so their `SKILL.md` descriptions point up to `sde-vs-esi` first. See [`skills/sde-vs-esi/docs/adr/0001`](./skills/sde-vs-esi/docs/adr/0001-front-door-invocation.md). The routing rule is **hard** where data is single-sourced and **advisory** in the overlap set (where both sources can answer) — it recommends, the caller decides.
