# SDE REST API

How to fetch the SDE from CCP's REST API at `https://sde.evetech.net` (generally available since 2026-11-19). CCP announced it in [The SDE over HTTP: static data, dynamically](https://developers.eveonline.com/blog/the-sde-over-http-static-data-dynamically). URLs and cache times are also in [`../manifest.yaml`](../manifest.yaml). Everything here was also checked against the live service.

## Layout

| URL | Returns |
|---|---|
| `/` | Tenants: `{"tenants":[{"name":"tranquility",…}]}` |
| `/tranquility/` | Releases, newest first, as `{release, releaseUrl}`. Only the current and previous release are kept. |
| `/tranquility/latest/` | 302 to the current release, e.g. `/tranquility/20261116T124006Z/`. Anything under `latest/` redirects the same way. |
| `/tranquility/<release>/` | The release index: `{release, buildNumber, releaseDate, tables}` |
| `/tranquility/<release>/<table>/<key>.json` | One record. A missing key returns 404 `{"error":"Not found"}`. |
| `/tranquility/<release>/<table>.jsonl` | The whole table, as a 302 to `/blobs/<version>.jsonl` |

Each `tables` entry has:

- `version`: a hash of the table. CCP says it changes only when the table's data changes, but on 2026-11-16 it also changed for `accountingEntryTypes` when only its schema changed. Treat any change as a reason to re-download.
- `jsonl.url` and `jsonl.recordSchemaUrl` (a JSON Schema for one record).
- `record.urlTemplate` (`<table>/{key}.json`), `record.schemaUrl` and `record.exampleUrl`.
- `flatbuffers.binaryUrl` (`.fb`), `flatbuffers.schemaUrl` (`.fbs`) and `flatbuffers.binarySchemaUrl` (`.bfbs`).

The release name, not `buildNumber`, identifies a version: two releases can share a build number (ADR 0003). Releases appear about 10 minutes after a Tranquility deployment, the same as the zip, and sometimes in between (for example when new military campaign objectives start). The API is served from a CDN and isn't affected by daily downtime.

## Fetching

- **Every blob is gzipped**, with `content-encoding: gzip` even when the request asks for `identity`. `curl` and `wget` save the compressed bytes unless you pass `curl --compressed`. Node's `fetch` and most HTTP libraries decompress for you.
- **Follow redirects.** Tables, record schemas and FlatBuffers files all 302 to `/blobs/…`. `curl` without `-L` saves a small redirect page.
- **Conditional requests:** send the `ETag` back in `If-None-Match`, and an unchanged resource returns `304 Not Modified`.
- **Cache times:** `latest/` and the release list `max-age=300`; the release index and table redirects `max-age=86400`; blobs and dated records `max-age=31536000, immutable` with `ETag: "<version>"`; 404s `max-age=60`.
- **CORS:** `access-control-allow-origin: *` comes back only when the request sends an `Origin` header, so test from a browser, not `curl`.

Sizes at release `20261116T124006Z` (gzip / raw): `types` 25 MB / 153 MB, `typeDogma` 1.3 MB / 24 MB, `mapSolarSystems` 1.1 MB / 4.8 MB. `mapMoons` is about 200 MB raw.

## Syncing

1. `HEAD /tranquility/latest/` and read the release name from `location`.
2. If it matches the release you hold, stop.
3. Otherwise fetch the release index and compare each table's `version` with the one you stored. Download only the tables that changed, from that release's own URLs (not `latest/`), so a release that lands mid-download can't mix two releases. Don't hold a release URL for long: it stops working two releases later.

Don't drive this from `_changes.jsonl`. It lists keys added since the previous release, not edited records: release `20261120T165826Z` changed records in 14 tables and its `_changes` was empty.

## Record shape

- **The key field is named.** Each record starts with its own key field, named by `x-sde-key` in the table's record schema: `typeID` for `types`, `groupID` for `groups`, `solarSystemID` for `mapSolarSystems`. There is no `_key`. Some names don't follow the table name: `planetResources` → `celestialID`, `blueprints` → `blueprintTypeID`, `skinLicenses` → `licenseTypeID`, `skinrSlotsToMaterials` → `factionID`, `freelanceJobSchemas` → `major`, `_sde` → `key`. [`datasets.md`](datasets.md) lists the key for each table.
- **Nested keyed lists are named too.** Where the zip writes `{"_key":…,"_value":…}` pairs, the API writes named fields. `masteries` 582 is `{"typeID":582,"levels":[{"level":0,"certificateIDs":[96,…]},…]}`; `typeBonus` writes `types[].{typeID, bonuses}`; `missions` messages are `{"kind":…,"text":{"en":…}}`.
- **Required fields are always present; optional fields with no value are left out.** `typeLists` always has all six lists, even empty ones. Optional empty lists such as `typeBonus.miscBonuses` are omitted (they were written as `[]` until release `20261120T165826Z`). Check the record schema's `required` list before relying on a key being there.
- **Numbers are plain JSON.** A mass of `1.0` is written `1`, so a typed parser must accept an integer where it expects a float. Very large values such as `massDust` on moons stay floats.
- **Missing numbers are 0** on the fields CCP's changelog marks "now also set when 0", the same as the zip. See the zero-fill trap in [`traps.md`](traps.md).
- **Meta tables:** `_sde` (`{"key":"sde","buildNumber":…,"releaseDate":…}`), `_changes` (`{"comparedWith":"tranquility@<release>","tables":[…]}`) and `_schemaChangelog` (CCP's schema changelog as records, keyed by `afterBuildNumber`).

## FlatBuffers

Every table also comes as FlatBuffers, generated from the same data. Tested at release `20261116T124006Z` with `flatc` 25.12.19 on `groups`, `planetResources`, `typeDogma`, `mapSolarSystems` and `types`: row counts, order and values match the JSONL.

- **Shape.** The root is `<Name>Table { rows: [<Name>] (required); }`, one vector holding the whole table. The key field is marked `(key)`, and rows are sorted by it, so you can binary search the vector or use `LookupByKey` in generated code over a memory-mapped file. Nested keyed lists such as `masteries.levels` are vectors of tables with their own `key` field.
- **Decoding.** The `.fb` has no file identifier, so `flatc` needs `--raw-binary`. Add `--defaults-json` to print zero and false values the way the JSONL does. The `.bfbs` works in place of the `.fbs`:

  ```sh
  curl -sSL --compressed -o groups.fb  "$R/groups.fb"
  curl -sSL --compressed -o groups.fbs "$R/groups.fbs"
  flatc --json --strict-json --raw-binary --defaults-json groups.fbs -- groups.fb
  ```

- **Optional numbers.** A field declared `x: int = null` keeps "absent" separate from 0, and is null exactly where the JSONL leaves the key out (2,712 `planetResources` rows have no `workforce`). A field declared without `= null` defaults to 0, so absent and 0 look the same, for example `planetResources.power`.
- **Doubles.** `flatc` prints doubles to 12 significant digits. The binary holds the full value; read doubles through generated code, not `flatc`'s JSON.
- **Enums** decode to names (`"instanceInteger"`) in `flatc`'s JSON, where the JSONL has the integer.
- **Size.** Smaller than JSONL raw, but often larger gzipped: `typeDogma` is 3.97 MB gzipped against 1.33 MB for JSONL. Prefer FlatBuffers for random access from disk, JSONL for download size.
