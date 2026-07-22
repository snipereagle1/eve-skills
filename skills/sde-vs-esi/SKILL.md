---
name: sde-vs-esi
description: Decide whether a piece of EVE Online data comes from ESI, the Static Data Export (SDE), or both — and how to join them. Use when you're unsure which source answers a question, when a task mixes live/character data with static reference data (names, topology, dogma, blueprints), or when an ESI response hands you bare IDs that need names. Consult this before committing to ESI or the SDE.
---

You're building against EVE Online and a question needs data. Two sources can answer: **ESI** (the live HTTP API — characters, markets, structures, anything that changes minute to minute) and the **Static Data Export** (**SDE** — the offline dataset of everything that only changes on a patch: type names, the universe map, dogma, blueprints). Pick wrong and you either hammer a rate-limited API for data that never changes, or stand up an ingestion pipeline you didn't need. This skill is the decision, and the join for when the answer is *both*.

For how to *use* each source correctly once you've picked it, defer to the [`esi`](../esi/SKILL.md) and [`sde`](../sde/SKILL.md) skills — this one only routes. The concrete list of endpoints that duplicate SDE datasets (the **overlap set**) and its version anchors live in [`manifest.yaml`](manifest.yaml); re-verify against its `last_verified` before trusting a specific route.

## Route by data kind first — most questions aren't a choice

Before weighing anything, check whether the data even *exists* on both sides. Usually it doesn't, and then there's nothing to decide:

- **Live / character / corporation / market / structure state → ESI, only ESI.** Assets, wallet, skills-trained, contracts, open market orders and prices, fleet, mail, a player structure's fuel or services, kills. The SDE has none of it. It changes constantly; a static dump could never carry it.
- **Static / reference / mechanical → the SDE, canonically.** Blueprint materials and outputs, reprocessing yields, the full dogma model (what an attribute *means*, what modifies what), the complete type tree, skill prerequisites, planetary schematics, the raw stargate graph. Some of this ESI cannot answer at all; the rest it answers one-ID-at-a-time behind the error budget.

Only when both sources genuinely carry the same data — the **overlap set** — do you actually have a decision. Everything else is dictated by where the data lives.

## In the overlap, the default depends on what you've already built

The overlap set is small and specific: **type / group / category names and hierarchies, the solar-system / constellation / region map, NPC stations, market groups, dogma attribute & effect definitions, factions / races / bloodlines.** ESI exposes all of it under `/universe/*`, `/dogma/*`, `/markets/groups`; the SDE has the same records offline. Here — and only here — you choose. The right choice turns on **integration state**, because the SDE carries an **adoption cost** (an ingestion pipeline to download, index, and refresh it) that ESI's just-make-a-request model doesn't:

- **If the SDE is already integrated / that data is already loaded → prefer the SDE, always.** It's free, unauthenticated, not metered against the error budget, and bulk-joinable in one local pass. Resolving 10,000 type names locally beats 10,000 conditional GETs by every measure.
- **If the SDE is *not* integrated → default to ESI**, and resolve on demand (for many IDs, one bulk call — see the manifest's `universe-names` route — not a loop). Standing up an ingestion pipeline for incidental lookups is usually more trouble than it's worth.
- **Override that default when the static data is core functionality or high-volume.** If naming / topology / dogma is on your hot path, or you're resolving IDs by the thousand, the adoption cost pays for itself — adopt the SDE.

This recommendation is **advisory**. Outside the overlap the source is forced; inside it, you weigh the trade-off and decide. The skill informs the call; it doesn't make it for you.

## The common answer is "both, joined"

Most real work isn't ESI *or* SDE — it's ESI *and* SDE, stitched on shared IDs. ESI returns the live layer as bare integers (`type_id`, `system_id`, `region_id`, `corporation_id`, …); the SDE (or ESI's own `/universe/*` copy) turns those integers into names, volumes, groups, and attributes. "Show my assets with names and volumes" = ESI for the asset list + the SDE to name and size each `type_id`. "Price a fitting" = the SDE for the item skeleton + ESI markets for the ISK. Treat **both-joined** as a first-class outcome, not a fallback.

How the two actually reconcile — the shared join keys, the ID→name resolution that is the workhorse of nearly every EVE tool, and the one caveat that the SDE build lags live — is in [`references/reconciliation.md`](references/reconciliation.md).

## Recipes for tasks that span both sources

Walkthroughs that name the exact ESI endpoints and SDE datasets for tasks whose path crosses the boundary — rendering assets with names, pricing a fitting: [`references/recipes.md`](references/recipes.md).
