# SDE zip downloads

The fallback to the REST API in [`api.md`](api.md). CCP still publishes the whole SDE as one zip per build "for now", and wants to retire it eventually (no date given). URLs are in [`../manifest.yaml`](../manifest.yaml).

## Downloads and syncing

- `…/static-data/eve-online-static-data-latest-jsonl.zip` 302-redirects to the immutable build-numbered zip, `…/static-data/tranquility/eve-online-static-data-<build>-jsonl.zip` (about 100 MB at build 3698585). A `-yaml.zip` is published too.
- `…/static-data/tranquility/latest.jsonl` is a one-line pointer: `{"_key":"sde","buildNumber":<N>,"releaseDate":…}`. It is cached `max-age=300`, and every resource supports `ETag` and `Last-Modified`.
- Sync loop: read the pointer (or `HEAD` the shorthand and read the redirect); if the build number is unchanged, do nothing; otherwise download and re-index.
- **The zip can lag the API.** The API also publishes releases between deployments and republishes builds, and the zip doesn't follow every one: on 2026-11-20 the API was at build 3724492 while the zip was still at 3720952. A build number check can't see a republish of the same build (ADR 0003). The zip also has no `_changes` or `_schemaChangelog` tables.

## Formats

- **Prefer JSON Lines (`-jsonl.zip`).** One JSON object per line, streamable, low-memory. YAML (`-yaml.zip`) has native integer keys but loads whole-file and blows up on the big datasets (see the YAML trap in [`traps.md`](traps.md)).

## Encoding rules

The zip encodes keys differently from the API:

- **Every record's ID is a `"_key"` field.** JSON object keys must be strings, so the record's ID (the API's named key field, e.g. `typeID`) is written as `_key`. `_sde`'s key is `_key` here and `key` in the API.
- **Nested integer-keyed maps are arrays of `{"_key":…, "_value":…}`**, sometimes several deep. Reconstruct the real map from those pairs. `masteries` 582 is `{"_key":582,"_value":[{"_key":0,"_value":[96,139,85,87,94]},…]}`, where the API writes `levels[].{level, certificateIDs}`.
- **Empty lists are left out**, even for fields the API always writes (e.g. all six `typeLists` lists).

Localized names, the zero-fill trap and everything in [`datasets.md`](datasets.md) apply to both sources; only the key names differ.
