# Reconciliation: how ESI and the SDE fit together

When the answer is **both** sources, you join them on shared integer IDs: ESI carries the live layer, the SDE carries the reference layer, and they meet at the same keys. This file is the join catalog. For which endpoint has which shape, read the `esi` skill and the OpenAPI spec; for which SDE dataset holds what, read the `sde` skill. This is only the *seam*.

## ID→name resolution is the workhorse

Almost every EVE tool does the same thing: ESI hands back the live state as **bare integers**, and something has to turn those integers into names, volumes, groups, and attributes. That something is the SDE (or, when the SDE isn't integrated, ESI's own `/universe/*` copy).

The IDs ESI returns bare, and where each resolves:

| ESI gives you | Resolve via SDE dataset | Resolve via ESI (no SDE) |
|---|---|---|
| `type_id` | `types` → name; `groupID`→`groups`→`categoryID`; `typeDogma` for stats; `typeMaterials` for reprocessing | `GET /universe/types/{type_id}` |
| `system_id` / `constellation_id` / `region_id` | `mapSolarSystems` / `mapConstellations` / `mapRegions` | `GET /universe/systems/{id}` etc. |
| `station_id` (NPC) | `npcStations` (name derived via `operationID`→`stationOperations`) | `GET /universe/stations/{station_id}` |
| `market_group_id` | `marketGroups` | `GET /markets/groups/{market_group_id}` |
| `attribute_id` / `effect_id` | `dogmaAttributes` / `dogmaEffects` | `GET /dogma/attributes/{id}` etc. |
| `faction_id` / `race_id` / `bloodline_id` | `factions` / `races` / `bloodlines` | `GET /universe/factions` etc. |
| `character_id` / `corporation_id` / `alliance_id` | **not in the SDE** | `POST /universe/names` |

Two rules make this cheap and correct:

- **Never resolve one ID per request in a loop.** If the SDE is integrated, resolve locally in one pass. If it isn't, batch through `POST /universe/names` (up to 1000 mixed IDs per call, no scope) rather than N× `GET /universe/types/{id}`. A per-ID loop burns the ESI error budget and is slow — see the `esi` skill on the error budget.
- **Player-owned IDs are ESI-only.** Characters, corporations, alliances, and *player structures* (`/universe/structures/{id}`, not `/universe/stations`) don't exist in the SDE at all. Only NPC stations reconcile offline. Don't try to name a citadel from the SDE.

## The shared join keys

The same key means the same thing on both sides — that's what makes the join sound:

- **`type_id`** — the universal spine. ESI assets, market orders, blueprints, kills all speak `type_id`; the SDE's `types` is keyed by it.
- **`system_id` / `constellation_id` / `region_id`** — the map hierarchy, identical IDs in ESI location fields and the SDE `map*` datasets.
- **`region_id`** doubles as the market region key: ESI's `/markets/{region_id}/orders` uses the same ID the SDE names in `mapRegions`.
- **`attribute_id` / `effect_id`** — dogma. ESI's `/dogma/*` and the SDE's `dogmaAttributes`/`dogmaEffects` define the same IDs; per-type values live in the SDE's `typeDogma` (or ESI's `/dogma/dynamic/items/...` for mutated modules).

## Caveat: the SDE build lags live

The SDE is a point-in-time snapshot rebuilt roughly each Tranquility deploy; ESI is always current. So when CCP ships new content, an ESI payload can reference a `type_id` (a just-released ship, a new structure) that your SDE build doesn't contain **yet**. In practice this is minor — most integrations refresh the SDE build on a schedule (startup-time or per-deploy) and tolerate brief staleness — but the reconciliation code should **degrade, not crash**, on an ID the SDE can't resolve: fall back to `POST /universe/names` (or show the raw ID) rather than throwing. This is a caveat on the resolution step, not a reason to prefer one source. See the `sde` skill for the build-number sync loop.
