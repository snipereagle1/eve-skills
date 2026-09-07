# ESI Traps

Non-obvious pitfalls where reading the endpoint's spec is not enough — the field is missing, the data is encoded in a string, the access rules aren't what they look like, or the spec on hand is for a different compatibility date. Each entry: the trap, why it bites, what to do.

## A route the spec documents can still `404`

`/meta/openapi.json` is served **per compatibility date**. Fetch it without `X-Compatibility-Date` and you get the `2020-01-01` spec, which still documents routes retired since. The `components.parameters.CompatibilityDate` enum names the date the spec you're holding was rendered at — it reads `2020-01-01` undated and `2026-08-18` when you fetch with that date, and the two specs list different routes. Read a route out of the old spec, call it under a recent date, and the `404` says nothing about why. `/sovereignty/map` is one: `200` at `2020-01-01`, `404` from `2026-05-19`, where a sovereignty rework replaced it with `/sovereignty/systems`. Fetch the spec at the date you send, check the enum matches, and use `/meta/changelog` — keyed by compat date, each change flagged `is_breaking` — for when a route was retired and what replaced it.

## A `403` can mean "wrong in-game role," not "wrong token"

The spec marks an endpoint as needing a scope; you grant the scope; you still get `403`. Corp- and structure-scoped endpoints (e.g. `/corporations/{id}/structures/`) *also* require the authenticated character to hold a specific in-game **role** (Station_Manager, Director, etc.). The token is valid and correctly scoped — the character just isn't allowed. **Refreshing the token will not fix a role-based 403.** Distinguish: `401` → token problem (refresh/re-auth); `403` → missing scope *or* missing role (nothing to refresh).

## Ansiblex jump gates: the destination is only in the name string

There is **no** dedicated Ansiblex endpoint and **no** destination-system field anywhere in ESI. An Ansiblex is just a structure with `type_id == 35841` (group 1408, "Upwell Jump Bridge"), and the structure record gives you only the system it physically sits in (`system_id`). The far-side system exists **only inside the auto-generated name string**. So topology has to be string-parsed out of a display name, and a manually renamed gate breaks that parse. The name format and the full walkthrough are in [`recipes.md`](recipes.md).

## The public structure list hides almost every structure

`/universe/structures/` (the anonymous list) returns **only** structures whose ACL is fully public. Most player structures — Ansiblexes always — are not, so they never appear. To see a private structure you need an authenticated character who can dock there, via `search` + `/universe/structures/{id}/`, or the owning corp's `/corporations/{id}/structures/`. Do not treat the public list as an enumeration of "all structures."

## ESI never exposes structure ACLs or standings

You can read that a structure exists and its services, but ESI gives you **no** way to read its access-control list or standings contents — only an opaque profile id. Any logic that needs "who can dock / who is blue" cannot be answered from ESI; it must come from in-game or be maintained out of band.

## Singularity is gone; `datasource` is being replaced by `X-Tenant`

Upgrade trap: older code targets the test server via `datasource=singularity`. The `datasource` enum is now **tranquility-only** and the new OpenAPI spec drops `datasource` entirely for an **`X-Tenant`** header (default `tranquility`). Code that hard-codes a Singularity datasource silently breaks. Production data is Tranquility only.
