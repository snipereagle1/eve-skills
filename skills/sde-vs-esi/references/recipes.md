# Recipes: tasks that span both sources

Walkthroughs where the answer is **both** ESI and the SDE, joined. Each names the exact ESI endpoints (read the `esi` skill + OpenAPI spec for their shapes and scopes) and the exact SDE datasets (read the `sde` skill for their layout). The point here is the *seam* between them.

## Render a character's assets with names and volumes

**Goal:** turn ESI's raw asset list — which is nothing but IDs and quantities — into a human-readable inventory.

1. **ESI, live layer:** `GET /characters/{character_id}/assets/` — scope `esi-assets.read_assets.v1`. Paginate with `X-Pages`. Each row is `{ item_id, type_id, location_id, quantity, ... }` — no names, no sizes.
2. **Reference layer — the join:** for each distinct `type_id`, resolve name + `volume` + `groupID`:
   - **SDE integrated (preferred):** local lookup in `types` (name, `volume`, `groupID`→`groups`→`categoryID`). One pass, no API cost.
   - **SDE not integrated:** batch the distinct `type_id`s through `POST /universe/names` (≤1000/call) for names; `GET /universe/types/{type_id}` for `volume` if you need it. Prefer adopting the SDE if you do this often — this is exactly the "core / high-volume" override.
3. **Resolve `location_id`:** a station/structure or another asset (nesting). NPC stations resolve via the SDE `npcStations` or `GET /universe/stations/{id}`; **player structures are ESI-only** — `GET /universe/structures/{structure_id}` (scope `esi-universe.read_structures.v1`), never the SDE.
4. Join and render. Degrade gracefully on any `type_id` the SDE build doesn't have yet (new content) — fall back to `POST /universe/names` or show the raw ID.

## Price a fitting

**Goal:** given a ship + modules, compute a market price. The item skeleton is static; the ISK is live.

1. **SDE, reference layer:** expand the fitting to a bill of materials — the ship `type_id` plus each module/charge `type_id` and quantity. All static, all from the SDE `types` (and `typeDogma` if you're validating slots). No ESI needed to know *what* is in the fit.
2. **ESI, live layer — the prices:** for each `type_id`, get market data. Region market: `GET /markets/{region_id}/orders/?type_id=...` (public; honor its cache + the token-bucket rate limit on this route — see the `esi` skill). Or aggregates where available. `region_id` is the same key the SDE names in `mapRegions`.
3. Join on `type_id`, sum. The static half never needs re-fetching; only the price half is live and cache-bound.

**Boundary check:** prices are **ESI-only** — the SDE has no market data. The SDE tells you the fit; ESI tells you the cost. Neither substitutes for the other.
