---
name: esi
description: Best-practice discipline for consuming EVE Online's ESI API (esi.evetech.net) correctly. Use when building or debugging ESI calls, handling SSO auth / scopes / tokens, hitting rate or error limits (420/429), caching (ETag/Expires), paginating, or choosing a compatibility date. If you're unsure whether ESI or the SDE is the right source for a piece of data, see the `sde-vs-esi` skill first.
---

**ESI** (the EVE Swagger Interface) is EVE Online's HTTP API. Agents get it wrong by default — hammering it for static data, ignoring the error budget, polling past cache timers. This skill is the discipline that prevents that. It never duplicates the spec: for any endpoint's shape, read the live OpenAPI spec (see `manifest.yaml`). The rules below are the knowledge the spec does *not* give you.

ESI is mid-migration. Everything here presents the **canonical** (current) mechanism first; a legacy form is noted only because you'll meet it upgrading an old app. Every volatile URL, header, and current-state fact lives in [`manifest.yaml`](manifest.yaml) — treat it as the source of truth and re-verify against its `last_verified` date before trusting a specific value.

## Reach for the SDE, not ESI, for static data

The single highest-leverage rule. Data that only changes on a game patch — type names, system/region/constellation topology, dogma attributes, market groups, icons — belongs to the **Static Data Export**, not ESI. Resolving 10,000 type names by calling `/universe/types/{id}` is an error-budget-burning anti-pattern; the SDE is one download, offline, free. Rule of thumb: **static/reference → SDE; dynamic/character/live/market → ESI.** The SDE's location and format are in `manifest.yaml`. When the two sources overlap and the call isn't obvious — or when you need to *join* them — the [`sde-vs-esi`](../sde-vs-esi/SKILL.md) skill owns that decision.

## Cache before you call

ESI tells you exactly how long every response is good for. Honor it.

- **`Expires`** — the resource is valid until this time. **Never poll faster than the cache timer**; a request before `Expires` returns identical data and wastes your budget.
- **`ETag`** + **`If-None-Match`** — send the previous `ETag` back; if unchanged, ESI returns **`304 Not Modified`** with no body. Use this for every repeated GET: it saves bandwidth and does not spend an error the way a wasted call can.
- **`Last-Modified`** is also emitted for conditional requests.

## Respect the error budget

ESI meters failures, and blowing the budget locks you out of *all* routes, not just the failing one. Two systems coexist per route (a route uses one or the other):

- **Legacy error limit** — up to 100 non-2xx/3xx responses per 60s window, then **`420`** on everything. Watch **`X-ESI-Error-Limit-Remain`**; when it nears zero, **stop and wait** `X-ESI-Error-Limit-Reset` seconds. Fix the *cause* of the errors — don't retry into the wall.
- **New token-bucket limit** — returns **`429`** with **`Retry-After`** (seconds) and `X-Ratelimit-*` headers. Honor `Retry-After`.

Backoff is not optional. The correct response to 420/429 is to slow down, not to retry immediately.

## Identify every request

ESI requires apps to identify themselves. Send a **`User-Agent`** with your **app name + version and a contact email** (browsers that can't set it use `X-User-Agent`; last resort is the `user_agent` query param). Anonymous mass traffic gets throttled or blocked.

## Pin a compatibility date

ESI now versions your *whole application* against a date, not per-route `/v1` numbers.

- Send **`X-Compatibility-Date: YYYY-MM-DD`** (or the `compatibility_date` query param) on every request. Future dates are rejected; the effective date rolls at 11:00 UTC downtime. CCP holds ≥1 year of backward compatibility.
- Pick the date you built/reviewed against and hold it stable; bump it deliberately when you adopt newer behavior. The value the skill last verified is in `manifest.yaml`.
- *Legacy (upgraders only):* old `/v1`, `/latest`, `/legacy` routes still work but receive no new endpoints. New routes are compat-date-only. Migrate off versioned URLs.

## Paginate to the scheme the endpoint documents

- **`X-Pages`** (most list routes): fetch `page=1`, read the `X-Pages` response header, loop `page` 2..N. `page` starts at 1.
- **Cursor** (newer routes): `limit` + opaque `before`/`after` tokens — treat tokens as opaque, walk until an empty page.

Never guess the scheme; check what the endpoint's spec entry declares.

## Auth: the model and its traps

The OAuth2/SSO handshake itself is well-covered by CCP's SSO docs (in `manifest.yaml`) — don't re-derive it. What bites you:

- **Access tokens are short-lived (~20 min).** Don't cache them long; refresh on demand.
- **Refresh tokens are per-character** and long-lived — one per authorized character/scope-set. Store them keyed by character.
- **Scopes are a space-delimited string.** An endpoint needs a specific scope; missing it fails at call time, not login.
- **Validate the JWT properly:** issuer is `login.eveonline.com` (accept the `https://` form too); audience must contain your `client_id` **and** the literal `"EVE Online"`.
- **`401` vs `403`:** 401 → token problem, refresh or re-auth. 403 → missing **scope** *or* missing in-game **role**; there is nothing to refresh. (See `references/traps.md`.)

## When the spec lies or under-documents

Non-obvious traps where reading the spec is not enough — endpoints whose real behavior surprises you: [`references/traps.md`](references/traps.md).

## Recipes for confusing multi-step tasks

Walkthroughs that name the exact endpoints and scopes for tasks the spec doesn't obviously map to (e.g. deriving Ansiblex jump-gate topology): [`references/recipes.md`](references/recipes.md).
