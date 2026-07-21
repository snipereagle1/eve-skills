# ESI Recipes

Walkthroughs for tasks the spec doesn't obviously map to. Each names the exact endpoints and scopes; for request/response shapes, read the endpoint in the OpenAPI spec (see `../manifest.yaml`).

## Resolve names for many IDs (types, systems, regions) — use the SDE first

**The trap:** looping `/universe/types/{id}` (etc.) for thousands of IDs burns the error budget and is slow.

1. For **static** names — type, group, category, system, constellation, region, dogma — download the **SDE** once (see `sde-docs` in the manifest) and resolve locally. This covers the overwhelming majority of "what is this ID called" needs.
2. Only when you must resolve **arbitrary mixed IDs at runtime** (e.g. names of characters/corps/alliances that aren't in the SDE), use ESI's bulk **`POST /universe/names/`** — one call resolves up to 1000 IDs of many categories. No scope required.
3. Never resolve one-ID-per-request in a loop when a bulk or offline path exists.

## Derive Ansiblex jump-gate topology (the network graph)

**Goal:** build the graph of who-connects-to-where for a corp/alliance's Ansiblex jump bridges. ESI has no topology endpoint; you assemble it. See [`traps.md`](traps.md) for why.

**Path A — richest, if you have corp access (recommended):**
1. `GET /corporations/{corporation_id}/structures/` — scope `esi-corporations.read_structures.v1`, character role **Station_Manager**. One call gives `structure_id`, `type_id`, `system_id`, `name`, `fuel_expires`, `services[]` (look for the Conduit/Jump-bridge service online/offline), and `state`.
2. Keep only `type_id == 35841` (Ansiblexes).
3. Parse each `name` on the delimiter `" » "` (space + U+00BB + space): left of it is the source system label, right (up to `" - "`) is the **destination** system. Match the source against `system_id` as a sanity check.
4. Build edges source→dest. `fuel_expires` and the service state tell you which gates are actually usable.

**Path B — topology only, no corp access:**
1. Authenticated character `search`: `GET /characters/{id}/search/?categories=structure&search=»` — scope `esi-search.search_structures.v1` — to find structures whose name contains the guillemet (only structures that character can dock at appear).
2. For each hit, `GET /universe/structures/{structure_id}/` — scope `esi-universe.read_structures.v1` — for `name` and `type_id`; filter to `35841` and parse the name as above.
3. This yields topology but **no** fuel/state, and only for structures that character can reach.

**Do not** use the public `/universe/structures/` list here — Ansiblexes are never fully-public, so it returns nothing useful (see `traps.md`).

## Poll a list endpoint without wasting the error budget

1. First fetch: no conditional header. Store the response's **`ETag`** and honor its **`Expires`**.
2. Do not request again before `Expires`.
3. Next fetch after `Expires`: send **`If-None-Match: <stored ETag>`**. A **`304`** means nothing changed — reuse cached data, spend almost nothing. A `200` carries fresh data and a new `ETag`.
4. If the list is paginated, read **`X-Pages`** on page 1 and loop `page` 2..N (or walk the cursor). Cache each page's ETag independently.
