---
name: sde-vs-esi
description: Decide whether a piece of EVE Online data comes from ESI, the Static Data Export (SDE), or both — and how to join them. Use when you're unsure which source answers a question, when a task mixes live/character data with static reference data (names, topology, dogma, blueprints), or when an ESI response hands you bare IDs that need names. Consult this before committing to ESI or the SDE.
---

You're building against EVE Online and a question needs data. Two sources can answer: **ESI** (the live HTTP API — characters, markets, structures, anything that changes minute to minute) and the **Static Data Export** (**SDE** — the offline dataset of everything that only changes on a patch: type names, the universe map, dogma, blueprints). Pick wrong and you either hammer a rate-limited API for data that never changes, or stand up an ingestion pipeline you didn't need. This skill is the decision, and the join for when the answer is *both*.

For how to *use* each source correctly once you've picked it, defer to the [`esi`](../esi/SKILL.md) and [`sde`](../sde/SKILL.md) skills — this one only routes. The concrete list of endpoints that duplicate SDE datasets (the **overlap set**) and its version anchors live in [`manifest.yaml`](manifest.yaml); re-verify against its `last_verified` before trusting a specific route.

## Route by data kind first — most questions aren't a choice

Before weighing anything, check whether the data even *exists* on both sides. Usually it doesn't, and then there's nothing to decide:

- **Live / character / corporation / market / structure state → ESI, only ESI.** Assets, wallet, skills-trained, contracts, open market orders and prices, fleet, mail, a player structure's fuel or services, kills. The SDE has none of it. It changes constantly; a static dump could never carry it.
- **Static / reference / mechanical → the SDE, canonically.** Type, group and category records, the universe map, NPC stations, blueprint materials and outputs, reprocessing yields, the full dogma model (what an attribute *means*, what modifies what), skill prerequisites, planetary schematics, the raw stargate graph. ESI no longer serves most of this (see below).

Only when both sources genuinely carry the same data — the **overlap set** — do you actually have a decision. Everything else is dictated by where the data lives.

## Static data now comes from the SDE, with or without a pipeline

Since compat date 2026-11-19 ESI no longer serves static data: the `/universe` routes for types, groups, categories, the map, celestials, NPC stations, factions, races, bloodlines, ancestries, graphics and schematics are gone (404). CCP moved that data to the SDE's REST API at `sde.evetech.net`, which serves one record per request. So the old trade-off — an SDE ingestion pipeline or ESI's just-make-a-request lookups — no longer exists for that data. The **SDE's adoption cost** is now only a question of *how* you use the SDE:

- **If the SDE is already integrated / that data is already loaded → resolve locally, always.** It's free, unauthenticated, not metered against the error budget, and bulk-joinable in one local pass.
- **If the SDE is *not* integrated → fetch records from the SDE REST API on demand** (`<table>/<id>.json`; see the [`sde`](../sde/SKILL.md) skill). It needs no pipeline and is served from a CDN, outside ESI's error budget. For names only, ESI's bulk `POST /universe/names` (the manifest's `universe-names` resolver) still names types, systems, stations, regions and factions, up to 1000 IDs per call.
- **Integrate the SDE when the static data is core functionality or high-volume.** If naming / topology / dogma is on your hot path, or you're resolving IDs by the thousand, fetch whole tables (`<table>.jsonl`) and sync them by release instead of looping record lookups.

The **overlap set** — data both sources still serve — is now small: **market groups** (`/markets/groups`) and **dogma attribute & effect definitions** (`/dogma/*`). There, the same rule applies: local SDE if integrated; otherwise either source works, and ESI is fine for incidental lookups.

## The common answer is "both, joined"

Most real work isn't ESI *or* SDE — it's ESI *and* SDE, stitched on shared IDs. ESI returns the live layer as bare integers (`type_id`, `system_id`, `region_id`, `corporation_id`, …); the SDE (local, or its REST API) turns those integers into names, volumes, groups, and attributes. "Show my assets with names and volumes" = ESI for the asset list + the SDE to name and size each `type_id`. "Price a fitting" = the SDE for the item skeleton + ESI markets for the ISK. Treat **both-joined** as a first-class outcome, not a fallback.

How the two actually reconcile — the shared join keys, the ID→name resolution that is the workhorse of nearly every EVE tool, and the one caveat that the SDE build lags live — is in [`references/reconciliation.md`](references/reconciliation.md).

## Recipes for tasks that span both sources

Walkthroughs that name the exact ESI endpoints and SDE datasets for tasks whose path crosses the boundary — rendering assets with names, pricing a fitting: [`references/recipes.md`](references/recipes.md).
