---
name: sde
description: How to read EVE Online's Static Data Export (SDE) — the offline dataset of static game data. Use when working with EVE game data by ID or name (types, groups, categories, dogma attributes/effects, blueprints, the solar-system/region map, market groups, factions, NPC stations), resolving IDs↔names offline, downloading/syncing the SDE, or decoding its JSON Lines format.
---

The **Static Data Export (SDE)** is EVE Online's offline dataset of everything that only changes on a game patch — item types, dogma attributes, the universe map, blueprints, market groups, factions. Agents avoid it because it looks opaque: dozens of files, numeric IDs everywhere, no friendly API. It isn't opaque once you have the map. This skill *is* that map — what the SDE contains, how the pieces join, and how to read it correctly. It never duplicates the field-level schema: for exact record fields, read the live data or CCP's `schema-changelog.yaml` (see [`manifest.yaml`](manifest.yaml)).

The SDE was rebuilt from scratch in 2025 and the new layout is **not** backwards compatible. Everything here describes the **current** build-numbered SDE; a one-line legacy note is at the end for people porting old code. Every volatile URL and current-state fact lives in [`manifest.yaml`](manifest.yaml) — treat it as the source of truth and re-verify against its `last_verified` before trusting a specific value.

## The SDE is a versioned snapshot you sync by build number

The single framing that makes everything else make sense. The SDE is **not** a live service — it's a dataset rebuilt on (roughly) every Tranquility deployment and stamped with a **build number**. You download it once, index it, and use it offline until the build changes.

- The current build is a one-line pointer: `…/tranquility/latest.jsonl` holds `{"_key":"sde","buildNumber":<N>,"releaseDate":…}`. The full download's URL contains that same build number.
- **Sync, don't re-download.** The latest-`.zip` shorthand 302-redirects to the immutable, build-numbered URL. All resources support **`ETag`** and **`Last-Modified`**; the build pointer is cached `max-age=300`. So the correct refresh loop is: check `latest.jsonl` (or `HEAD` the shorthand and read the redirect) → if the build number is unchanged, do nothing → only when it changes, download and re-index. Blindly re-pulling ~95 MB every run is the anti-pattern.
- Because it's a snapshot, it is *stale between builds by design* — that's the point, and why static data belongs here rather than in per-request API calls.

## The domain map

The SDE is ~80 datasets, but they cluster into a few domains threaded by a handful of join keys. Once you know the keys, the rest is navigation. Full per-dataset inventory is one hop away in [`references/datasets.md`](references/datasets.md).

- **Items & industry.** `types` is the spine of the whole item system — every ship, module, mineral, blueprint, and skill is a `type` keyed by `typeID` (the record's `_key`). A type carries `groupID` → `groups` → `categoryID` → `categories` (the *mechanical* classification: what a thing **is**), plus optional `marketGroupID` and `metaGroupID`. Reprocessing output is in `typeMaterials` (`materialTypeID`+`quantity`); manufacturing is in `blueprints`.
- **Dogma (mechanics).** A type's stats and behaviors live in `typeDogma` as `dogmaAttributes` (attributeID→value) and `dogmaEffects` (effectID). Those IDs are *defined* in `dogmaAttributes` / `dogmaEffects` — you **join** to get names and meaning. Two consequences worth knowing up front: **skill prerequisites are encoded as attributes** (182/183/184… = required skill typeID, 277/278/1285… = required level; rank is attribute 275), and **what-modifies-what lives in `dogmaEffects.modifierInfo`** (`modifiedAttributeID`, `modifyingAttributeID`, `operation`).
- **Universe / map.** `mapRegions` ⊃ `mapConstellations` ⊃ `mapSolarSystems`, each child carrying its parent's ID. `mapStargates` are the **graph edges** — each has a source `solarSystemID` and a `destination.solarSystemID`; adjacency (and routing) is built from those. Celestials (`mapPlanets`, `mapMoons`, `mapStars`, `mapAsteroidBelts`) hang off systems.
- **Market.** `marketGroups` is a **separate tree** from `groups` — a browse taxonomy linked by `parentGroupID`, *not* the mechanical group/category hierarchy. Don't conflate them.
- **Factions, NPCs & PvE.** `factions`, `races`, `bloodlines`, `npcCorporations`, `npcStations` (which belong to a `solarSystemID` and an owner corp; their *name* is derived via `operationID` → `stationOperations`), plus agents, missions, and dungeons.
- **Cosmetics & UI.** `skins` and the `skinr*` family, `icons`, `graphics`.

## Read the format correctly

Two encodings bite every consumer on the very first record — they're not surprises, they're prerequisites.

- **Prefer JSON Lines (`-jsonl.zip`).** One JSON object per line, streamable, low-memory. It's the format to default to. YAML (`-yaml.zip`) is also published and has native integer keys, but it loads whole-file and blows up on the big datasets (`mapMoons` alone is ~210 MB uncompressed) — reach for it only if you specifically want native int keys.
- **Integer-keyed maps are encoded as `_key`/`_value`.** JSON object keys must be strings, so every record's ID is a `"_key"` field, and any *nested* integer-keyed map becomes an array of `{"_key":…, "_value":…}` (sometimes nested several deep, e.g. `masteries`). Reconstruct the real map from those pairs; don't expect a normal object.
- **Localized names are `{"en":…, "de":…, …}` objects — but not everywhere.** `types`, `groups`, `categories`, `marketGroups`, `factions` name fields are localized objects (pick a language, fall back to `en`). But `dogmaAttributes`/`dogmaEffects` names are **plain strings**, and some records (e.g. `npcStations`) have **no** name at all. Check the field's shape; don't assume every `name` is a localized object.

## While developing: the SDE MCP server

For interactive exploration during development — "what's the typeID of X", "what boosts mining yield", "route between two systems" — the [`eve-sde-mcp`](https://github.com/snipereagle1/eve-online-sde-mcp) server indexes the SDE and answers by ID or name without you parsing anything. It's an **optional convenience for dev-time**, not a dependency: production apps consume the raw SDE directly, and every rule and recipe here is written for that.

## When the data surprises you

Non-obvious pitfalls where reading a record isn't enough — the `published` flag, the two independent hierarchies, the split dogma model, what the SDE deliberately *doesn't* contain, and more: [`references/traps.md`](references/traps.md).

## Recipes for multi-step tasks

Walkthroughs that name the exact datasets and join keys for tasks whose path isn't obvious — bulk ID↔name resolution, building a skill-training plan, routing the stargate graph, blueprint↔product, the market tree, and reverse-modifier lookups: [`references/recipes.md`](references/recipes.md).

## Legacy (upgraders only)

The pre-2025 SDE — the `sde.zip` with `bsd/` + `fsd/` + `universe/` folders, per-file YAMLs like `invTypes`/`typeIDs.yaml`, and `nameID` fields — was **removed** by the rework and has no coexistence path. Target the current build-numbered SDE described above. If you're porting old code, the mapping (folders → `map*` datasets, `nameID` → `name`, `types` split into `masteries`/`typeBonus`, etc.) is in CCP's rework dev blog, linked from `manifest.yaml`.
