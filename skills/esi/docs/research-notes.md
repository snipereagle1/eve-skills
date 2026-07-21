# ESI (EVE Swagger Interface) — Current State & Best Practices

Research date: **2026-07-21**. All facts traced to primary sources: the EVE Developers portal
(`https://developers.eveonline.com`), CCP dev blogs, the live ESI OpenAPI/Swagger specs, and the
SSO docs. Confidence tags: **confirmed-primary** / **inferred** / **unconfirmed**.

> ⚠️ **Major recent shifts (since ~2024/2025), read first:**
> - **Versioning:** Per-route `/v1 /v2 … /latest /legacy /dev` versioning replaced by a single
>   **compatibility-date** model (`X-Compatibility-Date` header). Old versioned routes still work but
>   **new routes are compat-date-only**.
> - **Spec format:** Migrated from Swagger 2.0 → **OpenAPI 3.1/3.0**. The old `swagger.json` specs
>   **stopped receiving updates on 1 August** and are slated for eventual removal. New canonical spec:
>   `https://esi.evetech.net/meta/openapi.json`.
> - **Swagger UI → API Explorer:** `esi.evetech.net/ui/` redirects to `developers.eveonline.com/api-explorer`.
> - **datasource → X-Tenant:** In the new OpenAPI spec the `datasource` query param is gone; tenant is
>   an `X-Tenant` header (default `tranquility`). **Singularity is no longer an available datasource.**
> - **SDE reworked:** brand-new, not-backwards-compatible SDE published at
>   `https://developers.eveonline.com/static-data/`, built every Tranquility deploy.
> - **Rate limiting:** A new per-group **token-bucket** rate-limit system (with `429` + `Retry-After`)
>   is being rolled out alongside the legacy `420` error-limit system.

---

## 1. Versioning / Compatibility Date  — confirmed-primary

- **Mechanism:** version the *whole application* against ESI at a point in time, instead of per-route `v` numbers.
- **Header:** `X-Compatibility-Date` — value is **ISO date `YYYY-MM-DD`**.
- **Query-param fallback:** `compatibility_date` (same value; for clients that can't set headers).
- **Semantics:** "This application's ESI implementation was updated or reviewed at this date – give me the API behavior as it was at that date."
- **Rules:**
  - ESI **rejects compatibility dates set in the future**.
  - The date **rolls over to the next day at downtime (11:00 UTC)** — i.e. the API "changes date at 11:00 UTC".
  - CCP aims to keep **at least one year of backwards compatibility**. Non-breaking additions (optional request params, new response fields/headers/enum values) ship *within* existing compat dates.
- **Currently valid value:** The live OpenAPI spec's `CompatibilityDate` parameter enum contains **only `2020-01-01`** (also the spec `info.version`). So as of 2026-07-21 the single accepted/baseline compat date is **`2020-01-01`**. *(Flag: enum has one value today; expect more dates as breaking changes accrue.)*
- **Relationship to old route versioning:**
  - CCP **merged all the `v`-versions — including `dev`, `latest`, `legacy` — together** over prior months.
  - **Existing versioned routes still work "for the foreseeable future."**
  - **All new routes support compatibility dates only** and are **not** exposed under versioned URLs.
  - Migration is **"entirely optional at this point"** — **no removal deadline announced** for versioned URLs. *(Flag: no sunset date given.)*
- **Base URL:** `https://esi.evetech.net` (unchanged; confirmed as the sole `servers[].url` in the OpenAPI spec). Legacy routes still under `https://esi.evetech.net/latest` etc.

Sources:
- https://developers.eveonline.com/docs/services/esi/overview/
- https://developers.eveonline.com/blog/changing-versions-v42-was-getting-out-of-hand
- https://esi.evetech.net/meta/openapi.json (spec `info.version` = 2020-01-01; `CompatibilityDate` enum; `servers`)

---

## 2. Error Limiting  — confirmed-primary

Two systems currently coexist:

**Legacy "error rate limit" (still active on most routes):**
- Allows **at most 100 non-2xx/3xx responses per minute (60 s window)**; after that ESI returns **`420`** on all routes.
- Headers:
  - `X-ESI-Error-Limit-Remain` — "errors left in this time frame".
  - `X-ESI-Error-Limit-Reset` — "seconds left until next time frame and errors reset to zero".
- When the limit is hit, **all your requests are automatically discarded until the end of the time frame**.
- **Correct client behavior:** watch `X-ESI-Error-Limit-Remain`; when it approaches zero, **stop / back off** until `X-ESI-Error-Limit-Reset` seconds elapse. Fix the source of the errors rather than hammering.

**New floating-window rate limit (rolling out per route group):**
- Status code **`429`** + `Retry-After` (seconds).
- Headers: `X-Ratelimit-Group`, `X-Ratelimit-Limit` (e.g. `150/15m`), `X-Ratelimit-Remaining`, `X-Ratelimit-Used`.
- Token cost per response: **2XX = 2, 3XX = 1, 4XX = 5, 5XX = 0** tokens.
- The two systems are **mutually exclusive per route** (a route uses either the new rate limit or the legacy error limit).

Sources:
- https://developers.eveonline.com/docs/services/esi/best-practices/ (error-limit headers)
- https://developers.eveonline.com/docs/services/esi/rate-limiting/ (420, 100/min, new token system, 429)

---

## 3. Caching  — confirmed-primary

- **`Expires`** — when the cached resource expires, i.e. when updated data should be available.
- **`Last-Modified`** — when the data was last updated in the cache.
- **`ETag`** (a hash of the content) + **`If-None-Match`** request header → server replies **`304 Not Modified`** when unchanged.
- **Best practice:** respect `Expires` — **do not poll faster than the cache timer**. Use `ETag`/`If-None-Match`; a `304` returns no body (saves bandwidth) and — importantly — a 304 does **not** count against your error budget the way a wasted error would, and avoids needless load.

Sources:
- https://developers.eveonline.com/docs/services/esi/best-practices/
- Spec confirms `ETag` / `LastModified` response headers and `If-None-Match` / `If-Modified-Since` request params in `components`.

---

## 4. Pagination  — confirmed-primary

Three schemes exist. Match the scheme the specific endpoint documents.

**X-Pages (classic, most list endpoints):**
- Response header **`X-Pages`** = total number of pages.
- Request query param **`page`** (integer, **starts at 1**).
- Iteration: fetch `page=1`, read `X-Pages`, then loop `page` from 2..X-Pages.

**Cursor-based (newer listing routes):**
- Query params: **`limit`** (max records, may return fewer), **`before`** and **`after`** (opaque cursor tokens).
- Response includes a cursor object with `before`/`after` tokens; **treat tokens as opaque**.
- Iterate `before` to walk toward older records until an empty list; use `after` to poll for new/modified records. Records ordered by last-modified/created date.

**From-ID pagination** — also referenced for some routes (e.g. large historical logs); detail page not captured.

Sources:
- https://developers.eveonline.com/docs/services/esi/pagination/x-pages/
- https://developers.eveonline.com/docs/services/esi/pagination/cursor-based/

---

## 5. Auth Model (conceptual)  — mixed

- **Access token TTL:** ~**20 minutes**; token response returns `expires_in` ≈ **1199–1200 s**. *(Flag: the SSO portal page states tokens are "time-limited" but does not print an exact TTL; the ~1200 s figure is from the token-endpoint response / long-standing docs — **inferred, strongly supported**.)*
- **Refresh tokens:** long-lived, **per-character** (one per authorized character/scope-set), used to mint new access tokens indefinitely until the user revokes access. — confirmed-primary
- **Scope string format:** **space-delimited** list (`scope=<space-separated list of scopes>`). — confirmed-primary
- **JWT validation:**
  - **Issuer (`iss`):** `https://login.eveonline.com` **or** `login.eveonline.com` (accept both). — confirmed-primary
  - **Audience (`aud`):** an array containing **your app's `client_id`** and the static value **`"EVE Online"`**. — confirmed-primary
  - JWKS / metadata via `https://login.eveonline.com/.well-known/oauth-authorization-server`. JWT `sub` = `CHARACTER:EVE:<character-id>`, `name` = character name, `scp` = granted scopes.
- **401 vs 403:** *(Flag: not spelled out on the primary SSO page — **inferred / common practice**.)* `401 Unauthorized` = missing/expired/invalid access token (re-auth or refresh). `403 Forbidden` = valid token but missing the required **scope**, or the character lacks the in-game **role** for that corp/structure resource.
- **App registration:** the **EVE Online Developers Portal** — `https://developers.eveonline.com/applications`.
- **Canonical SSO flow docs:** `https://developers.eveonline.com/docs/services/sso/`

Sources:
- https://developers.eveonline.com/docs/services/sso/
- Token-endpoint / JWT claim behavior corroborated by esi-docs SSO flow pages (docs.esi.evetech.net/docs/sso/).

---

## 6. User-Agent / Etiquette  — confirmed-primary

- Applications **must identify themselves**. Three mechanisms, in order:
  - **`User-Agent`** request header (standard clients).
  - **`X-User-Agent`** header (browser/JS apps that can't set `User-Agent`).
  - **`user_agent`** query param (last-resort fallback).
- Recommended contents (both **strongly preferred**): **a contact email address** and **app name + version**.
- General etiquette: respect caches (§3), respect the error/rate limits (§2), don't parallel-hammer, and use ETags.

Sources:
- https://developers.eveonline.com/docs/services/esi/best-practices/

---

## 7. Datasource / Tenant  — confirmed-primary (CHANGED)

- **Legacy Swagger spec:** `datasource` **query param**, `enum` now contains **only `tranquility`** (default `tranquility`). **`singularity` has been removed** from the enum — the SISI datasource is no longer offered via ESI. *(Flag: change vs older docs that listed `singularity`.)*
- **New OpenAPI spec:** no `datasource` query param at all; tenant is selected via **`X-Tenant`** header, **default `tranquility`**.
- Net: production data = Tranquility only; **do not rely on a Singularity datasource**.

Sources:
- `https://esi.evetech.net/latest/swagger.json` (`parameters.datasource.enum` = `["tranquility"]`)
- `https://esi.evetech.net/meta/openapi.json` (`components.parameters.Tenant` → `X-Tenant`, default `tranquility`)

---

## 8. SDE Boundary  — confirmed-primary (REWORKED)

- Guidance: **static/reference game data belongs in the SDE, not ESI.** The SDE "contains static game data that only changes with game updates."
- **New SDE (reworked, NOT backwards-compatible with the old S3 export):** published at **`https://developers.eveonline.com/static-data/`**, rebuilt **on every Tranquility deployment**.
  - Latest JSON Lines: `https://developers.eveonline.com/static-data/eve-online-static-data-latest-jsonl.zip`
  - Latest YAML: `https://developers.eveonline.com/static-data/eve-online-static-data-latest-yaml.zip`
  - (These redirect to build-numbered URLs.) Resources fully support **ETag** and **Last-Modified**.
- *(Flag: the old `bsd/` and `universe/` folder structure was removed; the old S3 `eve-static-data-export` bucket layout is superseded — code written against the pre-2025 SDE will break.)*

Sources:
- https://developers.eveonline.com/docs/services/static-data/
- https://developers.eveonline.com/blog/reworking-the-sde-a-fresh-start-for-static-data

---

## 9. Spec & UI Locations  — confirmed-primary (CHANGED)

- **New OpenAPI JSON (canonical, compat-date model):**
  - OpenAPI **3.1**: `https://esi.evetech.net/meta/openapi.json`
  - OpenAPI **3.0**: `https://esi.evetech.net/meta/openapi-3.0.json`
  - *(CCP: consider these "beta while we fine-tune them.")*
- **Legacy Swagger 2.0 JSON (route-versioned; frozen):**
  - `https://esi.evetech.net/latest/swagger.json` (also `/legacy/`, `/dev/`, and `_latest` variants)
  - **Stopped receiving updates on 1 August; scheduled for removal at a later (unannounced) date.** *(Flag.)*
- **Interactive UI / API Explorer (canonical):** `https://developers.eveonline.com/api-explorer`
  - Old Swagger UI `https://esi.evetech.net/ui/` **redirects** to the new API Explorer.

Sources:
- https://developers.eveonline.com/blog/changing-specs-from-swagger-to-openapi
- Live fetch of `https://esi.evetech.net/meta/openapi.json` (openapi 3.1.0; single server `https://esi.evetech.net`)

---

## Canonical resource table

| URL | Purpose | Kind | Compat date / verified |
|---|---|---|---|
| `https://esi.evetech.net` | ESI API base URL (all routes) | spec/api | verified 2026-07-21 |
| `https://esi.evetech.net/meta/openapi.json` | Current machine-readable spec (OpenAPI 3.1) | spec | version 2020-01-01; verified 2026-07-21 |
| `https://esi.evetech.net/meta/openapi-3.0.json` | Current spec, OpenAPI 3.0 variant | spec | verified 2026-07-21 |
| `https://esi.evetech.net/latest/swagger.json` | Legacy Swagger 2.0 spec (frozen, being removed) | spec | v1.36, tranquility-only; verified 2026-07-21 |
| `https://developers.eveonline.com/api-explorer` | Interactive API Explorer (replaces `esi.evetech.net/ui/`) | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/esi/overview/` | ESI overview: compat date, base concepts | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/esi/best-practices/` | Error limit, caching, User-Agent etiquette | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/esi/rate-limiting/` | New token-bucket rate limit + legacy 420 error limit | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/esi/pagination/x-pages/` | X-Pages pagination | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/esi/pagination/cursor-based/` | Cursor pagination (limit/before/after) | portal | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/sso/` | SSO / OAuth2 conceptual docs, JWT validation | sso-docs | verified 2026-07-21 |
| `https://developers.eveonline.com/applications` | App registration (client_id / scopes) | portal | verified 2026-07-21 |
| `https://login.eveonline.com/.well-known/oauth-authorization-server` | SSO OAuth metadata / JWKS discovery | sso-docs | verified 2026-07-21 |
| `https://developers.eveonline.com/docs/services/static-data/` | SDE docs + download shorthands | sde | verified 2026-07-21 |
| `https://developers.eveonline.com/static-data/eve-online-static-data-latest-jsonl.zip` | Latest SDE, JSON Lines | sde | verified 2026-07-21 |
| `https://developers.eveonline.com/static-data/eve-online-static-data-latest-yaml.zip` | Latest SDE, YAML | sde | verified 2026-07-21 |
| `https://developers.eveonline.com/blog/changing-versions-v42-was-getting-out-of-hand` | Dev blog: compat-date rollout | dev-blog | verified 2026-07-21 |
| `https://developers.eveonline.com/blog/changing-specs-from-swagger-to-openapi` | Dev blog: Swagger→OpenAPI migration | dev-blog | verified 2026-07-21 |
| `https://developers.eveonline.com/blog/reworking-the-sde-a-fresh-start-for-static-data` | Dev blog: SDE rework | dev-blog | verified 2026-07-21 |
| `https://docs.esi.evetech.net/` | Older esi-docs (SSO flow details); partly superseded | community/portal | verified 2026-07-21 |

## Open uncertainties / could-not-confirm-from-primary
- **Access-token TTL exact value** — SSO portal says "time-limited" without a number; ~1200 s taken from token-response `expires_in`. **Inferred.**
- **401 vs 403 exact causes** — not enumerated on the primary SSO page; stated here from standard OAuth/ESI behavior. **Inferred.**
- **Versioned-URL removal date** — CCP announced none. **Unconfirmed (none exists yet).**
- **"1 August" Swagger-freeze year** — blog says "from August 1st"; year not restated in the excerpt (contextually 2025). **Flag.**
- **Compat-date enum** currently lists only `2020-01-01`; whether additional dates exist behind the scenes not confirmable beyond the published spec. **Confirmed-primary as of today.**
