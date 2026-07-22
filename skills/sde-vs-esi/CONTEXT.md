# SDE-vs-ESI Skill

A distributable, model-invoked reference skill that owns the datasource decision the `esi` and `sde` skills defer: given a question about EVE Online data, does the agent hit ESI, the SDE, or both — and how do the two reconcile? It is the front door the sibling skills point *up* to. Routing discipline in the spine; join/reconciliation detail in references.

## Language

**Routing rule**:
The spine's decision procedure, producing three outcomes — ESI-only, SDE-only, or both-joined. It is **hard** where data is single-sourced (the caller is *told* the source, because the other side lacks the data) and **advisory** in the overlap set (it recommends; the caller decides). Data kind first, then the overlap recommendation keyed on integration state. The rot-resistant core, always loaded.
_Avoid_: datasource selection, decision matrix, which-to-use guide

**Overlap set**:
The reference data *both* sources can answer — type/system/group/dogma names and hierarchies (ESI's `/universe/*`, `/markets/groups/` vs. the SDE's datasets). The only place the routing rule is advisory rather than forced; everything outside it is single-sourced and non-negotiable.
_Avoid_: shared data, duplicated endpoints

**Adoption cost**:
The SDE's precondition that ESI lacks — standing up an ingestion pipeline to load and refresh the dataset. The reason the overlap default is a function of what the user has *already built*, not just the data kind.
_Avoid_: setup cost, integration overhead

**Integration state**:
Whether the caller already has the SDE ingested/loaded. If yes, the SDE is always preferred in the overlap set; if no, ESI is the default *unless* the static data is core functionality or high-volume, which justifies paying the adoption cost.
_Avoid_: SDE availability, environment

**Documentation manifest**:
The structured `manifest.yaml` enumerating this skill's one volatile external fact — the **overlap set**, the ESI routes that duplicate SDE datasets — each with `url`, `purpose`, `kind`, `last_verified`. Dual-anchored: valid only relative to a known `verified_compat_date` (ESI) *and* `build` (SDE), since a routing decision can move when either source's surface moves. Walkable for drift-checking.
_Avoid_: sources list, links, bibliography

**Reconciliation**:
The reference half of the skill — how the two sources fit together once both are in play. Core workhorse is ID→name resolution (ESI returns bare `type_id`/`system_id`/`region_id`/…, the SDE names and enriches them) plus the shared join keys; the SDE build lagging live is a caveat here, not its own trap. Disclosed on demand, not in the spine. `Recipe` and `Trap` carry the sibling skills' meanings.
_Avoid_: integration, join layer (as the skill's topic), glue code
