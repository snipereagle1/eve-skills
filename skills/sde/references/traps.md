# SDE Traps

Non-obvious pitfalls where reading a single record isn't enough — the meaning is split across files, encoded in an ID, or simply absent. Each entry: the trap, why it bites, what to do.

## Unpublished types pollute everything

Every `type` (and many other records) carries a `published` boolean, and a huge number of them are **`false`** — removed items, test/dev junk, unreleased content, placeholders like typeID 0 (`#System`). If you enumerate or search `types` without filtering `published == true`, you get a firehose of things no player can ever see, and name/ID lookups return duplicates and ghosts. **Filter to `published` for anything user-facing.** Keep unpublished only when you specifically need historical or internal data.

## `groups`/`categories` and `marketGroups` are two unrelated hierarchies

There are two completely independent taxonomies over the same types, and conflating them is a classic bug:

- **`groups` → `categories`** is the *mechanical* classification — what a thing **is** (a Rifter is in group Frigate, category Ship). Reached via `types.groupID`.
- **`marketGroups`** is the *browse taxonomy* — where a thing appears in the in-game Market window, a tree linked by `parentGroupID`. Reached via `types.marketGroupID`.

They do not line up: a type's marketGroup path is not derivable from its group/category, and vice versa. Pick the one that matches your question — "what kind of thing is this?" → group/category; "where would a player find this for sale?" → marketGroup.

## Dogma is split across three files, and the meaning is in the joins

A type's stats are **not** self-describing. `typeDogma` gives you `dogmaAttributes` as `{attributeID, value}` pairs and `dogmaEffects` as `{effectID}` — bare numbers. To make sense of them you must join:

- `attributeID` → **`dogmaAttributes`** for the name (a *plain string*, not localized), unit, and `highIsGood`.
- `effectID` → **`dogmaEffects`** for the effect name and — crucially — its **`modifierInfo`**, which is where "this effect modifies attribute X by attribute Y using operation Z" actually lives.

Two things people miss:

- **Skill prerequisites are dogma attributes, not a field.** There is no `requiredSkills: [...]` array — the prerequisite skill, its level, and the skill's own training rank are all numbered attributes you read out of the attribute soup. The attribute numbers and the walk are in the skill-plan recipe.
- **"What boosts attribute X" is a reverse lookup over `modifierInfo`.** Nothing indexes modifiers by the attribute they modify; you scan `dogmaEffects` and build that index yourself. See the reverse-modifier recipe.

## The SDE is not the whole game

The SDE is *static* data only. It deliberately does **not** contain:

- **Market prices** — no ISK values. Prices are live; get them from ESI/market APIs.
- **Anything dynamic or player-owned** — character sheets, player structures/citadels, corp assets, contracts, live sovereignty. Only NPC stations exist here (`npcStations`), never player structures.
- **Computed results** — `typeMaterials` is *base* reprocessing composition; the actual yield depends on ore quantity/portion size, reprocessing skills, implants, and station efficiency, none of which are in the SDE. Likewise dogma gives base attributes, not stacking-penalised fitted values. The SDE gives you the inputs; **you** do the math.

If a question needs prices, live state, or a simulated result, the SDE is the wrong source (or only half the answer).

## Blueprints hide everything under `activities`

A `blueprints` record has no top-level materials or product. Everything is nested under **`activities`**, keyed by activity name — `manufacturing`, `invention`, `copying`, `reaction`, `research_material`, `research_time` — and each activity that has them carries `materials[]`, `products[]`, `skills[]` (all `{typeID, quantity}`), and a `time`. Consequences:

- To find what a blueprint *makes*, read `activities.manufacturing.products` (or `reaction.products`) — not a top-level field.
- There is **no product→blueprint index.** To go from a product typeID back to its blueprint you must build the reverse map yourself by scanning blueprints' `products`. (See the blueprint recipe.)
- Not every blueprint has every activity; check the key exists before indexing it.

## YAML loads whole-file and blows up on the big datasets

The SDE ships in JSON Lines and YAML. YAML has native integer keys (no `_key`/`_value` dance), which is tempting — but YAML parsers load the entire document into memory, and several datasets are enormous: `mapMoons` is ~210 MB uncompressed, `types` ~144 MB, `missions` ~50 MB. Parsing those as YAML can exhaust memory or take minutes. **Default to JSON Lines and stream it line-by-line;** reach for YAML only for small datasets where native int keys are worth it.
