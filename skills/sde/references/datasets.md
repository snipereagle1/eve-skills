# SDE Datasets

The full inventory of the current build-numbered SDE, grouped by domain. Each entry: the JSONL filename, the one question it answers, and its join keys. This is the street-level map; the domain overview and the load-bearing relationships are in [`../SKILL.md`](../SKILL.md).

**Not a schema.** Field lists change per build — for exact fields read the live data or CCP's `schema-changelog.yaml` (see [`../manifest.yaml`](../manifest.yaml)). Every record is keyed by `_key` (its numeric ID); see the format rules in `SKILL.md` for the `_key`/`_value` and localized-name encodings. Verified against build **3441022** (79 datasets); a rework build may add or drop files, so treat this list as current-as-of `last_verified`, not eternal.

## Items & industry — the type system

| Dataset | Answers | Join keys |
|---|---|---|
| `types` | What is this item? (name, mass, volume, published, portionSize) | `_key`=typeID → `groupID`, `marketGroupID`, `metaGroupID` |
| `groups` | The mechanical group a type belongs to | `_key`=groupID → `categoryID` |
| `categories` | The broadest mechanical class (Ship, Module, Charge…) | `_key`=categoryID |
| `metaGroups` | Tech tier / meta variant label (T1, T2, Faction…) | `_key`=metaGroupID ← `types.metaGroupID` |
| `typeMaterials` | Reprocessing composition of a type (base, pre-efficiency) | `_key`=typeID; `materials[].materialTypeID` → `types` |
| `blueprints` | How a thing is manufactured/invented/reacted | `_key`=blueprintTypeID; `activities.*.{materials,products,skills}[].typeID` → `types` |
| `typeBonus` | Role/ship bonuses attached to a type | `_key`=typeID |
| `typeLists` | Named groupings of types (curated lists) | `_key`; entries → `types` |
| `typeElements` | Structured sub-elements of a type | `_key`=typeID |
| `masteries` | Certificate mastery tiers per type | `_key`=typeID; nested `_key`/`_value` by level → `certificates` |
| `certificates` | Skill-certificate definitions | `_key`=certificateID |
| `compressibleTypes` | Which types compress, and to what | `_key`=typeID → compressed `types` |
| `contrabandTypes` | Contraband status/fines by faction | `_key`=typeID → `factions` |
| `controlTowerResources` | POS tower fuel/resource requirements | `_key`=typeID |
| `planetSchematics` | Planetary-industry production recipes | `_key`=schematicID; inputs/outputs → `types` |

## Dogma — mechanics

| Dataset | Answers | Join keys |
|---|---|---|
| `typeDogma` | This type's attribute values and effects | `_key`=typeID; `dogmaAttributes[].attributeID` → `dogmaAttributes`; `dogmaEffects[].effectID` → `dogmaEffects` |
| `dogmaAttributes` | Definition of an attribute (name is a plain string, unit, highIsGood) | `_key`=attributeID → `dogmaAttributeCategories`, `dogmaUnits` |
| `dogmaAttributeCategories` | Grouping of attributes for display | `_key`=categoryID |
| `dogmaEffects` | Definition of an effect + `modifierInfo` (what it modifies) | `_key`=effectID; `modifierInfo[].{modifiedAttributeID,modifyingAttributeID,operation}` |
| `dogmaUnits` | Display units for attribute values | `_key`=unitID |
| `dynamicItemAttributes` | Mutaplasmid mutation ranges (abyssal modules) | `_key`=mutatorTypeID → `types` |
| `dbuffCollections` | Warfare-buff attribute bundles | `_key` → `dogmaAttributes` |

## Universe / map

| Dataset | Answers | Join keys |
|---|---|---|
| `mapRegions` | A region (name, faction) | `_key`=regionID |
| `mapConstellations` | A constellation | `_key`=constellationID → `regionID` |
| `mapSolarSystems` | A solar system (security, position, celestials) | `_key`=solarSystemID → `constellationID`, `regionID`; `stargateIDs`, `planetIDs`, `starID` |
| `mapStargates` | The graph edges between systems | `_key`=stargateID; `solarSystemID` (source) + `destination.solarSystemID` |
| `mapPlanets` | A planet | `_key`=planetID → `solarSystemID` |
| `mapMoons` | A moon (huge dataset — ~210 MB in YAML) | `_key`=moonID → `planetID`/`solarSystemID` |
| `mapStars` | A star | `_key`=starID → `solarSystemID` |
| `mapAsteroidBelts` | An asteroid belt | `_key`=beltID → `planetID`/`solarSystemID` |
| `mapSecondarySuns` | Secondary suns (visual) | `_key` → `solarSystemID` |
| `landmarks` | Named map landmarks | `_key`=landmarkID → position |

## Market

| Dataset | Answers | Join keys |
|---|---|---|
| `marketGroups` | A node in the market browse tree (**separate from `groups`**) | `_key`=marketGroupID → `parentGroupID`; `types.marketGroupID` points here |

## Factions, NPCs & PvE

| Dataset | Answers | Join keys |
|---|---|---|
| `factions` | A faction (Caldari State, etc.) | `_key`=factionID → `corporationID`, `militiaCorporationID`, `memberRaces`, `solarSystemID` |
| `races` | A playable race | `_key`=raceID |
| `bloodlines` | A bloodline | `_key`=bloodlineID → `raceID`, `corporationID` |
| `ancestries` | A character ancestry | `_key`=ancestryID → `bloodlineID` |
| `npcCorporations` | An NPC corp (agents, stations, LP store) | `_key`=corporationID → `factionID`, `ceoID`; nested `_value` divisions |
| `npcCorporationDivisions` | Division definitions used by NPC corps | `_key`=divisionID |
| `npcStations` | A station in space (**no name field** — derived) | `_key`=stationID → `solarSystemID`, `ownerID` (corp), `operationID` → `stationOperations`, `typeID` |
| `npcCharacters` | NPC characters (agents, etc.) | `_key`=characterID → `corporationID` |
| `agentTypes` | Agent-type enumeration | `_key`=agentTypeID |
| `agentsInSpace` | Agents that appear in space | `_key` → `npcCharacters`, `solarSystemID` |
| `corporationActivities` | NPC corp activity definitions | `_key` |
| `stationOperations` | Station operation → services + the station's display name | `_key`=operationID → `stationServices` |
| `stationServices` | Station service definitions | `_key`=serviceID |
| `missions` | Mission definitions | `_key`=missionID |
| `dungeons` | Dungeon/encounter definitions | `_key`=dungeonID |
| `epicArcs` | Epic-arc mission chains | `_key` → `missions` |
| `militaryCampaigns` | Faction-warfare campaigns | `_key` → `factions`, `solarSystemID` |
| `militaryCampaignObjectives` | Objectives within campaigns | `_key` → `militaryCampaigns` |
| `mercenaryTacticalOperations` | Mercenary-den tactical ops | `_key` |
| `freelanceJobSchemas` | Freelance-job (mission) schemas | `_key`; nested `_value` |
| `sovereigntyUpgrades` | Sovereignty structure upgrades | `_key`=typeID → `types` |
| `planetResources` | Planetary resource distributions | `_key` → planets/`types` |

## Cosmetics & UI

| Dataset | Answers | Join keys |
|---|---|---|
| `skins` | A ship SKIN (cosmetic) | `_key`=skinID → `types` (applicable hulls), `skinMaterials` |
| `skinLicenses` | The item that grants a SKIN | `_key`=licenseTypeID → `skinID` |
| `skinMaterials` | SKIN material/appearance definitions | `_key`=skinMaterialID |
| `skinrComponents` | SKINR customization components | `_key` → component categories/rarities |
| `skinrComponentCategories` | SKINR component categories | `_key` |
| `skinrComponentPointValues` | SKINR component point costs | `_key`/`_value` |
| `skinrComponentRarities` | SKINR component rarities | `_key` |
| `skinrSlots` | SKINR customization slots | `_key` |
| `skinrSlotCategories` | SKINR slot categories | `_key` |
| `skinrSlotConfigurations` | SKINR slot configurations | `_key` |
| `skinrSlotNames` | SKINR slot names | `_key` |
| `skinrTierThresholds` | SKINR tier thresholds | `_key`/`_value` |
| `graphics` | Graphic asset references for types | `_key`=graphicID |
| `graphicMaterialSets` | Graphic material sets | `_key` |
| `icons` | Icon asset references | `_key`=iconID ← e.g. `marketGroups.iconID` |

## Character progression & UI trees

| Dataset | Answers | Join keys |
|---|---|---|
| `characterAttributes` | The five character attributes (Int, Mem…) | `_key`=attributeID |
| `characterTitles` | Corp title definitions | `_key` |
| `cloneGrades` | Jump-clone grade definitions | `_key` |
| `shipTreeElements` | Nodes in the in-client ship tree | `_key` → `types`/`groups` |
| `shipTreeFactions` | Ship-tree faction groupings | `_key` → `factions` |
| `shipTreeGroups` | Ship-tree group groupings | `_key`/`_value` → `groups` |
| `archetypes` | Structure/entity archetypes | `_key` |

## Meta

| Dataset | Answers | Join keys |
|---|---|---|
| `_sde` | This build's number + release date | `{"_key":"sde","buildNumber":…,"releaseDate":…}` |
| `translationLanguages` | The language codes used in localized name objects | `_key`=language code |
