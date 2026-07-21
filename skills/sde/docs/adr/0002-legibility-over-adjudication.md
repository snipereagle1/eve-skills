# Legibility over adjudication; tool-agnostic spine

**Context.** Two framings were on the table for an SDE skill. One: an *adjudication* skill centered on "when do I use the SDE vs ESI?" — the datasource boundary. Two: a *legibility* skill centered on "what is in the SDE and how do I read it?" The observed pain point is developers looking at the SDE and thinking "I don't want to deal with this" — opacity, not indecision. Separately, a companion `eve-sde-mcp` server exists that indexes the SDE, raising whether the skill should assume it.

**Decision.**

1. **Legibility is the thesis.** The spine's job is to make the SDE navigable — the domain map, join keys, and format-decoding rules. The SDE-vs-ESI trade-off is **descoped to a future `sde-vs-esi` skill**; this skill states only its own half of the boundary (the SDE is static-only) in passing.
2. **The spine is tool-agnostic.** It teaches consuming the *raw* SDE, because that's what production apps do (they parse the JSONL themselves). The `eve-sde-mcp` server is named only as an *optional dev-time* convenience — one line in the spine, one `status: optional` entry in the manifest — and never woven into the disciplines or recipes.

## Considered Options

- **Adjudication-first (SDE vs ESI as the headline)** — rejected: doesn't cure opacity, and duplicates the boundary term the ESI skill already owns. Deferred to its own skill.
- **MCP-forward (assume the server as the access layer)** — rejected: makes the skill useless to the majority case (apps parsing the raw SDE) and couples a distributable reference to one tool.
- **Legibility thesis + tool-agnostic spine + MCP as optional pointer** — chosen.

## Consequences

- `CONTEXT-MAP.md`'s shared **SDE boundary** term points *forward* to the future `sde-vs-esi` skill rather than being expanded here; the ESI context keeps its half unchanged.
- Recipes describe raw-data walks with no MCP calls, so they run in any application; the MCP is a dev-time shortcut a reader may or may not have.
- If the `sde-vs-esi` skill is written later, it consumes both this skill's SDE half and the ESI skill's half of the boundary — neither needs to change.
