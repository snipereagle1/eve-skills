# SDE Datasets

The full inventory of the SDE tables, grouped by domain. Each entry: the table name (the same in the REST API and the zip), the one question it answers, and its join keys. This is the street-level map; the domain overview and the load-bearing relationships are in [`../SKILL.md`](../SKILL.md).

**Not a schema.** Field lists change per build — for exact fields read the live data or CCP's `schema-changelog.yaml` (see [`../manifest.yaml`](../manifest.yaml)). Join keys use the REST API's field names: each record's own key comes first and is named by the table schema's `x-sde-key` (see [`api.md`](api.md)). The zip names that key `_key` and encodes nested keyed lists as `_key`/`_value` pairs instead (see [`zip.md`](zip.md)). Verified against release `20261116T124006Z` (build 3698585): 104 tables, of which the zip has 102 (no `_changes` or `_schemaChangelog`). Tables come and go between builds, so read the release index for the current list.

## Items & industry — the type system

| Dataset | Answers | Join keys |
|---|---|---|
| `types` | What is this item? (name, mass, volume, packagedVolume, published, portionSize) | `typeID` → `groupID`, `marketGroupID`, `metaGroupID` |
| `groups` | The mechanical group a type belongs to | `groupID` → `categoryID` |
| `categories` | The broadest mechanical class (Ship, Module, Charge…) | `categoryID` |
| `metaGroups` | Tech tier / meta variant label (T1, T2, Faction…) | `metaGroupID` ← `types.metaGroupID` |
| `typeMaterials` | Reprocessing composition of a type (base, pre-efficiency) | `typeID`; `materials[].materialTypeID` → `types` |
| `blueprints` | How a thing is manufactured/invented/reacted | `blueprintTypeID`; `activities.*.{materials,products,skills}[].typeID` → `types` |
| `typeBonus` | Role/ship bonuses attached to a type | `typeID` |
| `typeLists` | Named groupings of types (curated lists) | `typeListID`; entries → `types` |
| `typeElements` | Structured sub-elements of a type | `typeID` |
| `masteries` | Certificate mastery tiers per type | `typeID`; `levels[].{level, certificateIDs}` → `certificates` |
| `certificates` | Skill-certificate definitions | `certificateID` |
| `compressibleTypes` | Which types compress, and to what | `typeID` → compressed `types` |
| `contrabandTypes` | Contraband status/fines by faction | `typeID` → `factions` |
| `controlTowerResources` | POS tower fuel/resource requirements | `typeID` |
| `planetSchematics` | Planetary-industry production recipes | `schematicID`; inputs/outputs → `types` |
| `industryActivities` | The six industry activities (Manufacturing, Copying…) | `activityID` |
| `industryAssemblyLines` | Assembly-line kinds and their base time/material multipliers | `assemblyLineID` → `activityID` |
| `industryInstallationTypes` | Which assembly lines a structure or station type offers | `typeID` → `types`; `assemblyLines[].assemblyLineID` → `industryAssemblyLines` |
| `industryModifierSources` | Which dogma attributes modify industry cost/time/material per activity | `typeID` → `types`; `<activity>.{cost,time,material}[].dogmaAttributeID` → `dogmaAttributes` |
| `industryTargetFilters` | Named category filters that industry modifiers apply to | `targetFilterID`; `categoryIDs` → `categories` |
| `metenoxMoonDrill` | Metenox Moon Drill cycle time, efficiency and fuel use | `typeID` → `types` |

## Dogma — mechanics

| Dataset | Answers | Join keys |
|---|---|---|
| `typeDogma` | This type's attribute values and effects | `typeID`; `dogmaAttributes[].attributeID` → `dogmaAttributes`; `dogmaEffects[].effectID` → `dogmaEffects` |
| `dogmaAttributes` | Definition of an attribute (name is a plain string, unit, highIsGood) | `attributeID` → `dogmaAttributeCategories`, `dogmaUnits` |
| `dogmaAttributeCategories` | Grouping of attributes for display | `categoryID` |
| `dogmaEffects` | Definition of an effect + `modifierInfo` (what it modifies) | `effectID`; `modifierInfo[].{modifiedAttributeID,modifyingAttributeID,operation}` |
| `dogmaUnits` | Display units for attribute values | `unitID` |
| `dynamicItemAttributes` | Mutaplasmid mutation ranges (abyssal modules) | `typeID` → `types` |
| `dbuffCollections` | Warfare-buff attribute bundles | `dbuffCollectionID` → `dogmaAttributes` |
| `appliedProximityEffects` | Warfare buffs a type applies to ships near it | `typeID` → `types`; `dbuffs[].dbuffID` → `dbuffCollections` |
| `systemDbuffEmitters` | Types that pulse warfare buffs across a system | `typeID` → `types`; `dbuffs[].dbuffID` → `dbuffCollections` |
| `systemWideEffects` | System-wide effects (e.g. wormhole environments) | `typeID` → `types`; `dbuffs[].dbuffID` → `dbuffCollections`; `eligibleTypeListID` → `typeLists` |
| `linkWithShip` | Types that link to a ship and apply buffs while linked | `typeID` → `types`; `dbuffs[].dbuffID` → `dbuffCollections` |
| `proximityTrap` | Types that trigger when a ship comes in range | `typeID` → `types`; `triggerFilterTypeListID` → `typeLists` |
| `fighterAbilities` | Fighter ability definitions | `fighterAbilityID` |
| `fighterAbilitiesByType` | The abilities in each fighter type's slots | `typeID` → `types`; `abilitySlot0..2.abilityID` → `fighterAbilities` |

## Universe / map

| Dataset | Answers | Join keys |
|---|---|---|
| `mapRegions` | A region (name, faction) | `regionID` |
| `mapConstellations` | A constellation | `constellationID` → `regionID` |
| `mapSolarSystems` | A solar system (security, position, celestials) | `solarSystemID` → `constellationID`, `regionID`; `stargateIDs`, `planetIDs`, `starID` |
| `mapStargates` | The graph edges between systems | `stargateID`; `solarSystemID` (source) + `destination.solarSystemID` |
| `mapPlanets` | A planet | `planetID` → `solarSystemID` |
| `mapMoons` | A moon (huge dataset — ~210 MB in YAML) | `moonID` → `planetID`/`solarSystemID` |
| `mapStars` | A star | `starID` → `solarSystemID` |
| `mapAsteroidBelts` | An asteroid belt | `asteroidBeltID` → `planetID`/`solarSystemID` |
| `mapSecondarySuns` | Secondary suns (visual) | `sunID` → `solarSystemID` |
| `landmarks` | Named map landmarks | `landmarkID` → position |

## Market

| Dataset | Answers | Join keys |
|---|---|---|
| `marketGroups` | A node in the market browse tree (**separate from `groups`**) | `marketGroupID` → `parentGroupID`; `types.marketGroupID` points here |

## Factions, NPCs & PvE

| Dataset | Answers | Join keys |
|---|---|---|
| `factions` | A faction (Caldari State, etc.) | `factionID` → `corporationID`, `militiaCorporationID`, `memberRaces`, `solarSystemID` |
| `races` | A playable race | `raceID` |
| `bloodlines` | A bloodline | `bloodlineID` → `raceID`, `corporationID` |
| `ancestries` | A character ancestry | `ancestryID` → `bloodlineID` |
| `npcCorporations` | An NPC corp (agents, stations, LP store) | `corporationID` → `factionID`, `ceoID`; `divisions[].divisionID` → `npcCorporationDivisions` |
| `npcCorporationDivisions` | Division definitions used by NPC corps | `divisionID` |
| `npcStations` | A station in space (**no name field** — derived) | `stationID` → `solarSystemID`, `ownerID` (corp), `operationID` → `stationOperations`, `typeID` |
| `npcCharacters` | NPC characters (agents, etc.) | `characterID` → `corporationID` |
| `agentTypes` | Agent-type enumeration | `agentTypeID` |
| `agentsInSpace` | Agents that appear in space | `agentID` → `npcCharacters`, `solarSystemID` |
| `corporationActivities` | NPC corp activity definitions | `activityID` |
| `stationOperations` | Station operation → services + the station's display name | `operationID` → `stationServices` |
| `stationServices` | Station service definitions | `serviceID` |
| `missions` | Mission definitions | `missionID` |
| `dungeons` | Dungeon/encounter definitions | `dungeonID` |
| `epicArcs` | Epic-arc mission chains | `epicArcID` → `missions` |
| `militaryCampaigns` | Faction-warfare campaigns | `campaignID` → `factions`, `solarSystemID` |
| `militaryCampaignObjectives` | Objectives within campaigns | `objectiveID` → `militaryCampaigns` |
| `mercenaryTacticalOperations` | Mercenary-den tactical ops | `dungeonID` |
| `freelanceJobSchemas` | Freelance-job (mission) schemas | `major`; `schemas[]` |
| `sovereigntyUpgrades` | Sovereignty structure upgrades | `typeID` → `types` |
| `planetResources` | Planetary resource distributions | `celestialID` → planets/`types` |
| `schools` | Character-creation schools | `schoolID` → `corporationID`, `raceID`; `startingStations` → `npcStations` |
| `schoolMap` | Which solar system each school is in | `schoolMapID` → `schoolID`, `solarSystemID` |
| `stationStandingsRestrictions` | Standing a faction requires before its stations offer a service | `id` (a factionID) → `factions`; `services[].{serviceID, standing}` → `stationServices` |
| `corporationRoles` | Player-corporation role definitions | `roleID` → `roleGroupIDs` |
| `corporationRoleGroups` | Groupings of corporation roles | `roleGroupID` |

## Cosmetics & UI

| Dataset | Answers | Join keys |
|---|---|---|
| `skins` | A ship SKIN (cosmetic) | `skinID` → `types` (applicable hulls), `skinMaterials` |
| `skinLicenses` | The item that grants a SKIN | `licenseTypeID` → `skinID` |
| `skinMaterials` | SKIN material/appearance definitions | `skinMaterialID` |
| `skinrComponents` | SKINR customization components | `componentID` → component categories/rarities |
| `skinrComponentCategories` | SKINR component categories | `categoryID` |
| `skinrComponentPointValues` | SKINR component point costs | `categoryID`; `rarities[].{rarityID, points}` |
| `skinrComponentRarities` | SKINR component rarities | `rarityID` |
| `skinrSlots` | SKINR customization slots | `slotID` |
| `skinrSlotCategories` | SKINR slot categories | `categoryID` |
| `skinrSlotConfigurations` | SKINR slot configurations | `id` |
| `skinrSlotNames` | SKINR slot names | `id` |
| `skinrTierThresholds` | SKINR tier thresholds | `infoBubbleGroupID`; `tiers[].{level, maxPoints}` |
| `skinrSlotsToMaterials` | Which SKINR material each slot uses, per faction | `factionID`; `slots[].{slotID, materialID}` |
| `graphics` | Graphic asset references for types | `graphicID` |
| `graphicMaterialSets` | Graphic material sets | `materialSetID` |
| `icons` | Icon asset references | `iconID` ← e.g. `marketGroups.iconID` |

## Character progression & UI trees

| Dataset | Answers | Join keys |
|---|---|---|
| `characterAttributes` | The five character attributes (Int, Mem…) | `attributeID` |
| `characterTitles` | Corp title definitions | `titleID` |
| `cloneGrades` | Jump-clone grade definitions | `cloneGradeID` |
| `shipTreeElements` | Nodes in the in-client ship tree | `elementID` → `types`/`groups` |
| `shipTreeFactions` | Ship-tree faction groupings | `factionID` → `factions` |
| `shipTreeGroups` | Ship-tree group groupings | `groupID` (a ship-tree group, not `groups`); `elements[].elementID` → `shipTreeElements` |
| `archetypes` | Structure/entity archetypes | `archetypeID` |
| `skillPlans` | In-client recommended skill plans | `skillPlanID` → `factionID`; `milestones[]`/`skillRequirements[]` `{typeID, level}` → `types` |
| `expertSystems` | Expert Systems (temporary skill grants) | `typeID` → `types`; `skillsGranted[]` `{typeID, level}` → `types` |

## Meta

| Dataset | Answers | Join keys |
|---|---|---|
| `_sde` | This build's number + release date | `{"key":"sde","buildNumber":…,"releaseDate":…}` |
| `translationLanguages` | The language codes used in localized name objects | `language` code |
| `_changes` | Keys added to each table since the previous release (API only; omits edited records) | `comparedWith` (`tranquility@<release>`) |
| `_schemaChangelog` | CCP's schema changelog as records (API only) | `afterBuildNumber` |
| `accountingEntryTypes` | Wallet journal entry types | `accountingEntryTypeID` |
| `notificationTypes` | In-game notification types | `notificationTypeID` |
