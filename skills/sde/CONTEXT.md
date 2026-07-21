# SDE Skill

A distributable, model-invoked reference skill that makes EVE Online's Static Data Export **legible** — what it contains, how the datasets join, and how to read its format. It is a knowledge artifact, not a procedure, and never duplicates the field-level schema (that lives in CCP's `schema-changelog.yaml` and the live data). It does **not** adjudicate SDE-vs-ESI trade-offs — that is deferred to a future `sde-vs-esi` skill.

The domain vocabulary (Type, Group, Category, Blueprint, DogmaAttribute, SolarSystem, MarketGroup, Faction, NpcStation, …) is shared with and kept consistent with [`eve-online-sde-mcp`'s `CONTEXT.md`](https://github.com/snipereagle1/eve-online-sde-mcp) — the MCP server that indexes this same data. The terms below are the ones the *skill's design* turns on.

## Language

**Build snapshot**:
The SDE as a point-in-time dataset stamped with a **build number** and rebuilt (roughly) every Tranquility deployment. The skill's version anchor: it pins what it verified to a build number, not a route/date version. The current build is the `buildNumber` in `latest.jsonl`.
_Avoid_: version, release, dump, snapshot (unqualified)

**Domain map**:
The always-loaded core of the skill — the grouping of the ~80 datasets into a few domains plus the join keys that thread them (`typeID`, `groupID`/`categoryID`, the map hierarchy, `parentGroupID`). The cure for SDE opacity. Stops at join keys; never descends to field lists.
_Avoid_: schema, ERD, data dictionary, catalog

**Decoding rule**:
A format prerequisite for reading *any* SDE record: the `_key`/`_value` integer-key encoding (JSON keys must be strings) and the `{"en":…}` localized-name objects (present on some datasets, plain strings on `dogmaAttributes`/`dogmaEffects`, absent on others). Lives in the spine because it bites the first record, not as a trap.
_Avoid_: quirk, gotcha, edge case

**Trap**:
A non-obvious SDE pitfall where reading one record isn't enough — the meaning is split across files, encoded in an ID, or deliberately absent (e.g. skill prereqs as dogma attributes; no market prices). Disclosed in `references/traps.md`, never in the spine.
_Avoid_: gotcha, bug, edge case

**Recipe**:
A short walkthrough for a multi-step SDE task that names the exact datasets and join keys for the raw-data walk (e.g. building a skill-training plan, routing the stargate graph). Tool-agnostic; disclosed in `references/recipes.md`.
_Avoid_: workflow, tutorial, guide

**Documentation manifest**:
The structured `manifest.yaml` enumerating every canonical URL and volatile fact, each with `url`, `purpose`, `kind`, `status`, and `last_verified`, plus the current `build`. The single place to update when a new SDE build ships; walkable for drift-checking.
_Avoid_: sources list, links, bibliography

**SDE boundary** *(shared with the ESI context — do not expand here)*:
The choice of datasource by data kind — static/reference data belongs to the SDE, dynamic/character/live/market data to ESI. This skill states only its own half ("the SDE is static-only; no prices, no live state"); the full trade-off analysis is deferred to the future `sde-vs-esi` skill. Kept consistent with the ESI context's definition.
_Avoid_: SDE-vs-ESI, datasource selection (as this skill's topic)
