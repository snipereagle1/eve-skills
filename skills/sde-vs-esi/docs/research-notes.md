# sde-vs-esi — verification trail

Research date: **2026-07-21**. This skill introduces no new primary facts about either source — it composes the `esi` and `sde` skills, both already grounded against live CCP docs on the same date. The one thing verified fresh here is the **overlap set**: that each ESI route claimed to duplicate an SDE dataset actually exists in the live spec.

Confidence tags: **confirmed-spec** (read from the live OpenAPI spec) / **confirmed-sibling** (present in the sibling skill's verified docs) / **inferred**.

## Overlap set — confirmed-spec

Pulled `https://esi.evetech.net/meta/openapi.json` (453,965 bytes, compat date `2026-07-21`) and enumerated every path under the claimed prefixes. All routes in `manifest.yaml`'s `overlap_set` were confirmed present:

- `GET /universe/types[/{type_id}]`, `/universe/groups[/{group_id}]`, `/universe/categories[/{category_id}]`
- `GET /universe/systems[/{system_id}]`, `/universe/constellations[/{constellation_id}]`, `/universe/regions[/{region_id}]`
- `GET /universe/stargates/{stargate_id}`, `/universe/planets/{planet_id}`, `/universe/moons/{moon_id}`, `/universe/stars/{star_id}`, `/universe/asteroid_belts/{asteroid_belt_id}`
- `GET /universe/stations/{station_id}` (NPC stations; player structures are `/universe/structures/{id}` — **not** overlap)
- `GET /universe/factions`, `/universe/races`, `/universe/bloodlines`
- `GET /universe/graphics[/{graphic_id}]`, `/universe/schematics/{schematic_id}`
- `GET /dogma/attributes[/{attribute_id}]`, `/dogma/effects[/{effect_id}]`
- `GET /markets/groups[/{market_group_id}]`

Spec paths carry **no trailing slash**; the skill uses that form. (Both forms resolve on ESI, but the spec is the source of truth.)

## Bulk resolvers — confirmed-spec

- `POST /universe/names` and `POST /universe/ids` both present. `POST /universe/names` accepts up to **1000** IDs per call, no scope (limit carried over from the `esi` skill's verified recipes; the resolver is the fallback when the SDE isn't integrated).

## SDE dataset names — confirmed-sibling

Every `sde_dataset` referenced in `manifest.yaml` and `references/reconciliation.md` was grep-confirmed present in the sibling `sde` skill's `references/datasets.md` (build 3441022): `types, groups, categories, mapSolarSystems, mapConstellations, mapRegions, mapStargates, mapPlanets, mapMoons, mapStars, mapAsteroidBelts, npcStations, stationOperations, marketGroups, dogmaAttributes, dogmaEffects, typeDogma, factions, races, bloodlines, planetSchematics, graphics, icons`.

## Dual anchor — confirmed-sibling

`meta.verified_compat_date: 2026-07-21` mirrors `skills/esi/manifest.yaml`; `meta.build: 3441022` mirrors `skills/sde/manifest.yaml`. A routing recommendation is valid only relative to both; re-verify whenever either sibling bumps its anchor.

## Boundary facts leaned on (confirmed-sibling)

- **ESI-only** (no SDE equivalent): character/corp/alliance data, wallet, assets, contracts, live market orders & prices, player structures, mail, fleet, kills.
- **SDE-only** (ESI can't fully answer, or only one-ID-at-a-time behind the error budget): blueprint materials/products (`blueprints`, `typeMaterials`), full dogma model incl. `modifierInfo`, skill prerequisites (dogma attrs 182/183/184 + 277/278/1285, rank 275), the raw stargate adjacency graph, complete type tree.
