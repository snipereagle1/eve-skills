# Currency via build-number anchor and a documentation manifest

**Context.** The SDE was rebuilt from scratch in 2025 into a new, not-backwards-compatible, build-numbered dataset published every Tranquility deployment. Its shape (datasets, fields) changes over time, and CCP publishes a `schema-changelog.yaml` and a per-build `changes/<build>.jsonl` feed. A distributable reference skill that froze today's field lists in prose would be wrong within a build and would rot invisibly.

**Decision.** The skill body (`SKILL.md`) and reference files state only rot-resistant *legibility* — the domain map, the join keys, the format-decoding rules. Every volatile fact (download URLs, the current build number, format facts, schema/changes URLs) lives in a single structured `manifest.yaml`, and the skill anchors its verification to the SDE **build number** (read from `latest.jsonl`) rather than a date or route version. Maintenance is "re-verify the manifest against its build," not "hunt through prose." This mirrors the ESI skill's compat-date ADR, with the build number playing the role the compat date plays there.

## Considered Options

- **Snapshot field-level schema in prose** — rejected: duplicates CCP's `schema-changelog.yaml`, rots every build, and drags a distributed skill's credibility down.
- **Evergreen-only, no dataset specifics** — rejected: too abstract to cure the opacity that is the skill's entire reason to exist; readers need the dataset names and join keys.
- **Domain map (join-key altitude) in prose + volatile facts in a manifest** — chosen: the map is the rot-resistant legibility payload; the manifest quarantines everything that changes per build to one walkable file.

## Consequences

- The dataset inventory (`references/datasets.md`) is *semi*-volatile: it changes at rework-scale events (a build that adds/drops datasets), not every build. It's kept out of the always-loaded spine and updated when reworks land — which is when the skill would be revised anyway.
- `SKILL.md` almost never needs editing; drift is contained to `manifest.yaml` (URLs, `build`, `last_verified`) and occasionally `datasets.md`.
- The build number, not a date, is the re-verification key: bump `meta.build` from `latest.jsonl`, refetch the `sources`, update `last_verified`.
