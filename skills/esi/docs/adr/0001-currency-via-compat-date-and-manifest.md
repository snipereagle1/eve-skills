# Currency via compat-date anchor and a documentation manifest

**Context.** ESI is under active, breaking change: as of 2026 CCP has shipped compatibility-date versioning, migrated Swagger→OpenAPI, replaced the `datasource` param with an `X-Tenant` header (removing singularity), rebuilt the SDE, and begun rolling out a token-bucket rate limiter alongside the legacy error-limit one. A distributable reference skill that snapshots today's mechanics in prose would be wrong within a patch and would rot invisibly.

**Decision.** The skill body states only rot-resistant, HTTP-level *discipline*. Every volatile fact (spec URLs, portal locations, SSO/SDE endpoints, current mechanics) lives in a single structured `manifest.yaml`, and the skill anchors its verification to an ESI **compat date** rather than to route versions. Maintenance is "re-verify the manifest against its compat date," not "hunt through prose."

## Considered Options

- **Snapshot in prose with an "as of" banner** — rejected: scattered volatile facts rot invisibly and drag a distributed skill's credibility down.
- **Evergreen-only, no version specifics** — rejected: too abstract to be useful; readers need to know where the spec lives and roughly how versioning works.
- **Manifest + generated human view** — rejected: the generation step is sediment risk for little gain at this size.

## Consequences

- ESI is mid-migration, so the skill must document *dualities* (legacy 420 error-limit **and** new 429 token bucket; `/v1`-style routes **and** compat-date; `datasource` **and** `X-Tenant`) until CCP removes the old paths — the manifest's `compat_date`/`last_verified` fields are how a maintainer knows which half is current.
- Keeping volatile facts out of the spine means the body almost never needs editing; drift is contained to one walkable file.
