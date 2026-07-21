# EVE SDE — Current State & Structure (verification trail)

Research date: **2026-07-21**. Facts traced to primary sources: CCP's static-data docs/portal, the SDE rework dev blog, and — for everything about structure — a **full download of the live JSON Lines SDE** (build **3441022**, ~94 MB zip) extracted and inspected directly. Confidence tags: **confirmed-download** (checked against the real files) / **confirmed-primary** (CCP docs) / **inferred**.

> No assumptions: the dataset inventory and every record shape below were read out of the actual build-3441022 files, not from the docs (whose `schema-changelog.yaml` list was *incomplete* — it omitted `categories`, `agentsInSpace`, `contrabandTypes`, `controlTowerResources`, `dogmaAttributeCategories`, `skinLicenses`, `sovereigntyUpgrades`, all of which are present in the download).

## 1. Distribution, formats, versioning — confirmed-primary + confirmed-download

- **Formats:** JSON Lines (`-jsonl.zip`) and YAML (`-yaml.zip`). JSONL is one object per line, streamable; YAML has native int keys but loads whole-file.
- **Shorthand (latest) URLs** — confirmed via HEAD:
  - `https://developers.eveonline.com/static-data/eve-online-static-data-latest-jsonl.zip`
  - `…-latest-yaml.zip`
  - The shorthand **302-redirects** to the immutable build-numbered URL: `…/static-data/tranquility/eve-online-static-data-3441022-jsonl.zip`.
- **Build pointer:** `…/tranquility/latest.jsonl` → `{"_key":"sde","buildNumber":3441022,"releaseDate":"2026-07-21T11:06:27Z"}` (confirmed by fetch). The in-zip `_sde.jsonl` carries the same.
- **Schema changelog:** `…/tranquility/schema-changelog.yaml` — latest schema build **3407448** (2026-06-25), i.e. the schema trails the data build.
- **Changes feed:** `…/tranquility/changes/<build>.jsonl`.
- **Caching** — confirmed via HEAD on `latest.jsonl`: `ETag: "c972…"`, `Last-Modified: Tue, 21 Jul 2026 11:23:01 GMT`, `Cache-Control: max-age=300`. Docs state all resources support ETag/Last-Modified.
- **Cadence:** rebuilt with (roughly) every Tranquility deployment, behind a manual-approval gate — confirmed-primary (rework blog).

## 2. Inventory — confirmed-download

Build 3441022 contains **79 `.jsonl` datasets** (flat, no subdirectories). Full grouped inventory with join keys is in [`../references/datasets.md`](../references/datasets.md). Largest files: `mapMoons` 213.6 MB, `types` 144.2 MB, `missions` 50.9 MB, `mapPlanets` 48.6 MB, `typeDogma` 26.3 MB, `mapAsteroidBelts` 20.7 MB (uncompressed) — the basis for the "YAML blows up" trap.

## 3. Encodings — confirmed-download

- **`_key`:** every record's numeric id is the string field `"_key"` (JSON keys must be strings). Verified across all files.
- **`_key`/`_value` nested maps:** integer-keyed *nested* maps become arrays of `{"_key",…,"_value":…}`, sometimes several deep. Real example from `masteries`:
  `{"_key":582,"_value":[{"_key":0,"_value":[96,139,85,87,94]}, …]}`.
  Files containing `_value`: `masteries, npcCorporations, typeBonus, typeElements, missions, stationOperations, shipTreeGroups, shipTreeFactions, races, freelanceJobSchemas, skinrComponentPointValues, skinrTierThresholds` (grep-confirmed).
- **Localized names are inconsistent** — confirmed-download by checking `.name` type per dataset:
  - **Localized object** (`{"en","de",…}`): `types, groups, categories, marketGroups, factions, npcCorporations, races`.
  - **Plain string:** `dogmaAttributes, dogmaEffects`.
  - **No name field:** `npcStations` (name derived via `operationID` → `stationOperations`).

## 4. Key record shapes — confirmed-download

- `types` (Rifter 587): `{_key:587, groupID:25, marketGroupID:64, metaGroupID:1, published:true, name:{…}}`. → group→category, marketGroup, metaGroup joins.
- `groups`: `{_key, categoryID, name, published, …}`.
- `marketGroups`: `{_key, parentGroupID, hasTypes, iconID, name}` — tree via `parentGroupID`; **independent of** groups/categories.
- `typeDogma`: `{_key, dogmaAttributes:[{attributeID,value}], dogmaEffects:[{effectID,isDefault}]}`.
- `dogmaEffects` w/ modifier: `{_key:21, name:"shieldCapacityBonusOnline", modifierInfo:[{domain,func,modifiedAttributeID:263,modifyingAttributeID:72,operation:2}]}`.
- **Skill prereqs are dogma attributes** — confirmed on live type 12042 (Ares): attr **182**=`requiredSkill1`(→3328 Gallente Frigate), **277**=`requiredSkill1Level`(5), **183/278**=second skill. Skill **rank** is attr **275** (confirmed on skill type 3328: `275`=2.0, and its own prereq `182`=3327).
- `typeMaterials`: `{_key, materials:[{materialTypeID, quantity}]}` (note `materialTypeID`, not `typeID`).
- `blueprints`: `{_key, blueprintTypeID, maxProductionLimit, activities:{manufacturing:{materials[],products[],skills[],time}, invention:{…probability on products…}, copying:{time}, research_material:{time}, research_time:{time}, reaction?}}`. No top-level product; no product→blueprint index.
- `mapSolarSystems`: `{_key, constellationID, regionID, securityStatus, stargateIDs[], planetIDs[], starID, position, …}`.
- `mapStargates`: `{_key, solarSystemID (source), destination:{solarSystemID,stargateID}, typeID, position}` → adjacency for routing.
- `npcStations`: `{_key, solarSystemID, ownerID, operationID, typeID, reprocessing…}` — no name.
- `factions`: `{_key, corporationID, militiaCorporationID, memberRaces, solarSystemID, name, …}`.

## 5. What the SDE does NOT contain — confirmed-primary/inferred

No market prices, no character/player-structure/dynamic data (only NPC stations), no computed yields (reprocessing/fitting math is base inputs only). Prices and live state come from ESI.

## 6. Legacy (rework break) — confirmed-primary

The 2025 rework is not backwards compatible: removed `bsd/` and `universe/` folders (universe folded into `map*`), dropped client-unused fields, renamed `nameID`→`name`, split `masteries`/`typeBonus` out of `types`, added `agentTypes`/`dbuffCollections`/`dogmaUnits`/`dynamicItemAttributes`. Old S3 `sde.zip` layout superseded. Source: rework dev blog (in `manifest.yaml`).

## Open uncertainties
- **Exact community-schemas URL** — referenced by the static-data portal but not captured verbatim; manifest points at the portal page. **Flag.**
- **Legacy S3 URL** in the manifest is the historically-canonical old location, marked `status: legacy`; not re-verified live (expected gone). **Inferred.**
- **`reaction` activity** presence in `blueprints` — documented and expected; the sampled blueprints showed manufacturing/invention/copying/research_* activities. **Confirmed-download (partial).**
