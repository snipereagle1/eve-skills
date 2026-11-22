# sde-vs-esi — verification trail

Research date: **2026-07-21**. This skill introduces no new primary facts about either source — it composes the `esi` and `sde` skills, both already grounded against live CCP docs on the same date. The one thing verified fresh here is the **overlap set**: that each ESI route claimed to duplicate an SDE dataset actually exists in the live spec.

Confidence tags: **confirmed-spec** (read from the live OpenAPI spec) / **confirmed-sibling** (present in the sibling skill's verified docs) / **inferred**.

## Overlap set — confirmed-spec

Pulled `https://esi.evetech.net/meta/openapi.json` (453,965 bytes, compat date `2026-07-21`) and enumerated every path under the claimed prefixes. All routes in `manifest.yaml`'s `overlap_set` were confirmed present:

- `GET /universe/types[/{type_id}]`, `/universe/groups[/{group_id}]`, `/universe/categories[/{category_id}]`
- `GET /universe/systems[/{system_id}]`, `/universe/constellations[/{constellation_id}]`, `/universe/regions[/{region_id}]`
- `GET /universe/stargates/{stargate_id}`, `/universe/planets/{planet_id}`, `/universe/moons/{moon_id}`, `/universe/stars/{star_id}`, `/universe/asteroid_belts/{asteroid_belt_id}`
- `GET /universe/stations/{station_id}` (NPC stations; player structures are `/universe/structures/{id}` — **not** overlap)
- `GET /universe/factions`, `/universe/races`, `/universe/bloodlines`, `/universe/ancestries`
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

## Update 2026-11-21: ESI compat date 2026-11-19 removes the static /universe routes

Checked live against compat date `2026-11-19` and build `3724492`:

- The ESI changelog's `2026-11-19` entry lists 25 GET routes with `type: removed`: every overlap route under `/universe` (types, groups, categories, systems, constellations, regions, stargates, planets, moons, stars, asteroid belts, NPC stations, factions, races, bloodlines, ancestries, graphics, schematics). `GET /universe/types/587` returns 404 at `2026-11-19` and 200 at `2026-08-18`.
- Still served (200): `/dogma/attributes[/{id}]`, `/dogma/effects/{id}`, `/markets/groups[/{id}]`, `/universe/structures`, `/universe/system_jumps`, `POST /universe/names`. `POST /universe/names` still names `inventory_type`, `solar_system`, `station`, `region` and `faction` IDs.
- The SDE REST API serves the same records one at a time: `types/587.json`, `mapSolarSystems/30000142.json`, `factions/500001.json` and `npcStations/60003760.json` all return 200 under `latest/`.

So the overlap set shrinks to market groups and dogma definitions, and the "SDE not integrated" default moves from ESI to the SDE REST API's record lookup. The removed routes are listed under `removed_from_esi` in `manifest.yaml`.
