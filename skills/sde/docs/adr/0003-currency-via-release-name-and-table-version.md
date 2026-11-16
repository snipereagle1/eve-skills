# Currency via release name and per-table version

**Status:** accepted. Supersedes [ADR 0001](0001-currency-via-build-number-and-manifest.md)'s choice of the build number as the version anchor; its manifest decision stands.

**Context.** CCP now serves the SDE as a REST API at `sde.evetech.net` (Early Access as of 2026-11-16, with the zip expected to be deprecated). The API publishes **releases** named by timestamp (e.g. `20261116T124006Z`), and several releases can share one build number: `20261113T110647Z` and `20261116T124006Z` are both build 3698585, the second a republish where only table schemas changed. Under ADR 0001 a consumer that compares build numbers would miss that republish. Each table in a release also carries a `version` hash, which changes whenever that table's data or schema changes.

**Decision.** The **release name** identifies an SDE version, and the skill's manifest anchors to it (`meta.release`), keeping `meta.build` as a label. Consumers sync by following `latest/` to the release name and, when it changed, downloading only the tables whose `version` changed. The API is the default source and the zip the fallback.

## Considered Options

- **Keep the build number, add a caveat** — rejected: the build number can stay the same across a real change, so the decision itself was wrong, not just incomplete.
- **Amend ADR 0001 in place** — rejected: the reasoning that led to the build number was sound for the zip-only world and is worth keeping as history.
- **Release name as the version, per-table `version` for what to download** — chosen.

## Consequences

- `manifest.yaml` gains `meta.release`; `skills/sde-vs-esi` still mirrors `meta.build`, which keeps working.
- The `manifest-refresher` agent checks the release name from the `latest/` redirect, not `latest.jsonl`.
- `_changes.jsonl` is not a reliable sync signal: it does not list schema-only changes, while `version` does.
- Only the current and previous releases are kept, so a consumer can't pin an old release for long.
