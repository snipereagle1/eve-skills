# SDE zip downloads

The fallback to the REST API in [`api.md`](api.md). CCP still publishes the whole SDE as one zip per build, but expects to deprecate it. URLs are in [`../manifest.yaml`](../manifest.yaml).

## Downloads and syncing

- `…/static-data/eve-online-static-data-latest-jsonl.zip` 302-redirects to the immutable build-numbered zip, `…/static-data/tranquility/eve-online-static-data-<build>-jsonl.zip` (about 100 MB at build 3698585). A `-yaml.zip` is published too.
- `…/static-data/tranquility/latest.jsonl` is a one-line pointer: `{"_key":"sde","buildNumber":<N>,"releaseDate":…}`. It is cached `max-age=300`, and every resource supports `ETag` and `Last-Modified`.
- Sync loop: read the pointer (or `HEAD` the shorthand and read the redirect); if the build number is unchanged, do nothing; otherwise download and re-index.
- **The zip lags the API.** At build 3698585 the zip carries the 2026-11-13 release, while the API's current release is a 2026-11-16 republish of the same build. A build number check can't see that republish (ADR 0003). The zip also has no `_changes` or `_schemaChangelog` tables.

## Formats

- **Prefer JSON Lines (`-jsonl.zip`).** One JSON object per line, streamable, low-memory. YAML (`-yaml.zip`) has native integer keys but loads whole-file and blows up on the big datasets (see the YAML trap in [`traps.md`](traps.md)).

## Encoding rules

The zip encodes keys differently from the API:

- **Every record's ID is a `"_key"` field.** JSON object keys must be strings, so the record's ID (the API's named key field, e.g. `typeID`) is written as `_key`. `_sde`'s key is `_key` here and `key` in the API.
- **Nested integer-keyed maps are arrays of `{"_key":…, "_value":…}`**, sometimes several deep. Reconstruct the real map from those pairs. `masteries` 582 is `{"_key":582,"_value":[{"_key":0,"_value":[96,139,85,87,94]},…]}`, where the API writes `levels[].{level, certificateIDs}`.
- **Empty lists are left out**, where the API writes `[]`.

Localized names, the zero-fill trap and everything in [`datasets.md`](datasets.md) apply to both sources; only the key names differ.
