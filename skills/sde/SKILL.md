---
name: sde
description: How to read EVE Online's Static Data Export (SDE) — the offline dataset of static game data. Use when working with EVE game data by ID or name (types, groups, categories, dogma attributes/effects, blueprints, the solar-system/region map, market groups, factions, NPC stations), resolving IDs↔names offline, downloading/syncing the SDE, decoding its JSON Lines format, or using CCP's SDE REST API at sde.evetech.net.
---

The **Static Data Export (SDE)** is EVE Online's offline dataset of everything that only changes on a game patch — item types, dogma attributes, the universe map, blueprints, market groups, factions. Agents avoid it because it looks opaque: dozens of files, numeric IDs everywhere, no friendly API. This skill is the map — what the SDE contains, how the pieces join, and how to read it correctly. It never duplicates the field-level schema: for exact record fields, read the live data or CCP's `schema-changelog.yaml` (see [`manifest.yaml`](manifest.yaml)). When a question mixes static data with live or character state — or you're unsure which source owns it — the [`sde-vs-esi`](../sde-vs-esi/SKILL.md) skill owns that decision.

The SDE was rebuilt from scratch in 2025 and the new layout is **not** backwards compatible. Everything here describes the **current** SDE; a one-line legacy note is at the end for people porting old code. Every volatile URL and current-state fact lives in [`manifest.yaml`](manifest.yaml) — treat it as the source of truth and re-verify against its `last_verified` before trusting a specific value.

## The SDE is a versioned snapshot you sync by release

The single framing that makes everything else make sense. The SDE is **not** live data — it's a dataset rebuilt on (roughly) every Tranquility deployment. CCP publishes each rebuild as a **release** named by timestamp (e.g. `20261116T124006Z`) on the REST API at `sde.evetech.net`. You fetch what you need, keep it, and use it offline until the release changes.

- **Get it from the REST API** (Early Access; CCP expects to deprecate the zip). It serves one record (`<table>/<id>.json`) or one whole table (`<table>.jsonl`) per request, so you never need the full dataset to answer a lookup. Layout, gzip handling and FlatBuffers: [`references/api.md`](references/api.md).
- **Sync, don't re-download.** `HEAD …/tranquility/latest/` and read the release name from the redirect. If it's unchanged, stop. Otherwise read the release index and download only the tables whose `version` hash changed. Blindly re-pulling every table each run is the anti-pattern.
- **Version by release name, not build number.** Two releases can share a build number (a republish that changes only schemas), so a build check misses real changes.
- **The zip is the fallback** — one ~100 MB file per build, with a different key encoding. See [`references/zip.md`](references/zip.md).

## The domain map

The SDE is ~100 tables, but they cluster into a few domains threaded by a handful of join keys. Once you know the keys, the rest is navigation. Full per-dataset inventory is one hop away in [`references/datasets.md`](references/datasets.md).

- **Items & industry.** `types` is the spine of the whole item system — every ship, module, mineral, blueprint, and skill is a `type` keyed by `typeID`. A type carries `groupID` → `groups` → `categoryID` → `categories` (the *mechanical* classification: what a thing **is**), plus optional `marketGroupID` and `metaGroupID`. Reprocessing output is in `typeMaterials` (`materialTypeID`+`quantity`); manufacturing is in `blueprints`.
- **Dogma (mechanics).** A type's stats and behaviors live in `typeDogma` as `dogmaAttributes` (attributeID→value) and `dogmaEffects` (effectID). Those IDs are *defined* in `dogmaAttributes` / `dogmaEffects` — you **join** to get names and meaning.
- **Universe / map.** `mapRegions` ⊃ `mapConstellations` ⊃ `mapSolarSystems`, each child carrying its parent's ID. `mapStargates` are the **graph edges** — each has a source `solarSystemID` and a `destination.solarSystemID`; adjacency (and routing) is built from those. Celestials (`mapPlanets`, `mapMoons`, `mapStars`, `mapAsteroidBelts`) hang off systems.
- **Market.** `marketGroups` is the in-game Market browse taxonomy — a tree linked by `parentGroupID`, reached via `types.marketGroupID`.
- **Factions, NPCs & PvE.** `factions`, `races`, `bloodlines`, `npcCorporations`, `npcStations` (which belong to a `solarSystemID` and an owner corp; their *name* is derived via `operationID` → `stationOperations`), plus agents, missions, and dungeons.
- **Cosmetics & UI.** `skins` and the `skinr*` family, `icons`, `graphics`.

## Read the format correctly

Two encodings bite every consumer on the very first record — they're not surprises, they're prerequisites.

- **Each record's key field is named, and the name varies by table.** It comes first in the record and is given by `x-sde-key` in the table's record schema: `typeID`, `groupID`, `solarSystemID` — but `celestialID` for `planetResources` and `blueprintTypeID` for `blueprints`. [`references/datasets.md`](references/datasets.md) lists each table's key. The zip instead calls it `_key` and writes nested keyed lists as `_key`/`_value` pairs; see [`references/zip.md`](references/zip.md).
- **Localized names are `{"en":…, "de":…, …}` objects — but not everywhere.** `types`, `groups`, `categories`, `marketGroups`, `factions` name fields are localized objects (pick a language, fall back to `en`). But `dogmaAttributes`/`dogmaEffects` names are **plain strings**, and some records (e.g. `npcStations`) have **no** name at all. Check the field's shape; don't assume every `name` is a localized object.

## While developing: the SDE MCP server

For interactive exploration during development — "what's the typeID of X", "what boosts mining yield", "route between two systems" — the [`eve-sde-mcp`](https://github.com/snipereagle1/eve-online-sde-mcp) server indexes the SDE and answers by ID or name without you parsing anything. It's an **optional convenience for dev-time**, not a dependency: production apps consume the raw SDE directly, and every rule and recipe here is written for that.

## When the data surprises you

Non-obvious pitfalls where reading a record isn't enough — the `published` flag, the two independent hierarchies, the split dogma model (where skill prerequisites and what-modifies-what actually live), what the SDE deliberately *doesn't* contain, and more: [`references/traps.md`](references/traps.md).

## Recipes for multi-step tasks

Walkthroughs that name the exact datasets and join keys for tasks whose path isn't obvious — bulk ID↔name resolution, building a skill-training plan, routing the stargate graph, blueprint↔product, the market tree, and reverse-modifier lookups: [`references/recipes.md`](references/recipes.md).

## Legacy (upgraders only)

The pre-2025 SDE — the `sde.zip` with `bsd/` + `fsd/` + `universe/` folders, per-file YAMLs like `invTypes`/`typeIDs.yaml`, and `nameID` fields — was **removed** by the rework and has no coexistence path. Target the current SDE described above. If you're porting old code, the mapping (folders → `map*` datasets, `nameID` → `name`, `types` split into `masteries`/`typeBonus`, etc.) is in CCP's rework dev blog, linked from `manifest.yaml`.
