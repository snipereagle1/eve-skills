# SDE Recipes

Walkthroughs for tasks whose path through the data isn't obvious. Each names the exact datasets and join keys; for exact record fields, read the live data (see [`../manifest.yaml`](../manifest.yaml)). These are tool-agnostic — they describe the walk over the raw JSONL so they work in any application. (For interactive dev-time poking, the `eve-sde-mcp` server does several of these in one call, but nothing here depends on it.)

Throughout: every record is keyed by `_key` (its numeric ID); build id→record and name→record indexes over the file once, then seek.

## Resolve names ↔ IDs in bulk, offline

**The point of the whole skill.** Resolving thousands of static IDs to names (or back) by hitting a live API per ID is slow and burns rate/error budget. The SDE answers it locally.

1. Load the relevant dataset(s) — `types` for items, `groups`/`categories` for classification, `mapRegions`/`mapConstellations`/`mapSolarSystems` for places, `factions`, etc.
2. Build two indexes as you stream: `id → record` (from `_key`) and `lowercased-name → id`. Remember names are **localized objects** on most datasets (`name.en`, …) but **plain strings** on a few (`dogmaAttributes`, `dogmaEffects`) and **absent** on others (`npcStations`) — index the field in whatever shape it actually has.
3. Resolve in either direction against the in-memory index. For user-facing output, filter `published == true` so removed/test types don't surface.

This covers the overwhelming majority of "what is this ID called / what's the ID for this name" needs without a single network call.

## Build a skill-training plan (prerequisite tree + SP)

**Goal:** given a target type (ship/module) or skill, produce the full ordered list of prerequisite skills and the SP to train them. The prerequisites are buried in dogma attributes (see traps).

1. For the target typeID, read its `typeDogma`. Pull the **required-skill** attributes: `182/183/184/…` give the prerequisite **skill typeIDs**; the paired `277/278/1285/…` give the **required level** of each.
2. **Recurse:** for each required skill typeID, read *its* `typeDogma` and repeat — skills have prerequisite skills too. Collect the full transitive set; dedupe, keeping the highest required level when a skill appears more than once.
3. **Topologically sort** so every skill comes after its prerequisites.
4. **SP math:** each skill's training **rank** is dogma attribute **275**. SP required for level L at rank R is `R × 250 × √32^(L−1)` (i.e. levels 1–5 cost 250, 1 415, 8 000, 45 255, 256 000 SP × rank). Sum across the plan for total SP; carry a running cumulative if you want a timeline.

## Route between two solar systems (stargate graph)

**Goal:** shortest jump path A → B. There's no route dataset — you build the graph from stargates.

1. Stream `mapStargates`. Each record has a source `solarSystemID` and `destination.solarSystemID`. Add an adjacency edge `source → destination` (stargates are directional records but come in matched pairs, so the graph is effectively undirected).
2. Optionally enrich nodes from `mapSolarSystems` (e.g. `securityStatus` to prefer high-sec, or to weight jumps).
3. **BFS** over the adjacency map from A to B for fewest jumps (or Dijkstra with a security/danger weight). Return the ordered `solarSystemID` path and the jump count; resolve names via `mapSolarSystems`.

## Blueprint ↔ product

**Goal:** go from a blueprint to what it makes, or from a product back to the blueprint that makes it.

- **Blueprint → product:** read the `blueprints` record and look under `activities.manufacturing.products` (or `activities.reaction.products` for reactions) — each `{typeID, quantity}`. Inputs are the sibling `materials[]`; required skills are `skills[]`; duration is `time`. Not every blueprint has every activity — check the key exists.
- **Product → blueprint:** there is **no** reverse index. Build one once: stream `blueprints`, and for each, map every `activities.*.products[].typeID` → this `blueprintTypeID`. Then look the product typeID up in that map. (A product can, rarely, come from more than one blueprint/activity — keep a list.)

## Walk the market-group tree

**Goal:** the browse path for an item, or all items under a market node.

1. `marketGroups` is a tree linked by `parentGroupID` (root nodes have none). Build `id → record` and follow `parentGroupID` upward to get the root-to-node ancestry (the breadcrumb a player sees in the Market window).
2. To list items in a market group, filter `types` by `marketGroupID`. Note `marketGroups.hasTypes` tells you whether a node holds items directly or is only a container for child groups.
3. Keep this strictly separate from `groups`/`categories` — see the two-hierarchies trap.

## "What boosts attribute X" (reverse-modifier lookup)

**Goal:** find every effect (and thus every module/ship/skill) that modifies a given attribute — e.g. what increases mining yield (attribute 77), armor HP (265), etc.

1. Stream `dogmaEffects`. For each, scan its `modifierInfo[]` for an entry whose **`modifiedAttributeID`** equals your target attribute X. Collect those effectIDs; the entry also tells you the **`modifyingAttributeID`** (the source stat) and the **`operation`** (add, multiply, percentage…).
2. To get from effects back to the **things** that carry them, build a reverse index from `typeDogma`: `effectID → [typeIDs]` by scanning every type's `dogmaEffects`. Intersect with the effectIDs from step 1.
3. Resolve attribute IDs/names via `dogmaAttributes` (plain-string names). This is how you answer "which modules/ships affect this stat" without any prose parsing — it's all in `modifierInfo`.
