# The sde-vs-esi skill is the front door; the siblings defer up to it

**Context.** This skill's whole job is to fire *before* an agent commits to a datasource. But `esi` and `sde` are model-invoked on the presence of their own API — so left alone, one of them fires the moment the agent forms the intent "I'll hit ESI," and the routing decision never gets made. A routing skill that can't reliably run first is dead weight.

**Decision.** Make `sde-vs-esi` the entry point and have the two siblings **defer up** to it: their `SKILL.md` descriptions each carry a one-line "if unsure whether ESI or the SDE answers this, see `sde-vs-esi` first" pointer, and both `CONTEXT.md` files turn their old "deferred to a *future* `sde-vs-esi` skill" language into a live cross-reference. The skill stays a normal model-invoked skill (same shape as its siblings); it wins the race because the siblings point at it, not because of a broader trigger.

## Considered Options

- **Broad self-trigger only** ("fires whenever you need EVE data and haven't picked a source") — rejected: too fuzzy to fire reliably, and it competes with the siblings for the same moment instead of preceding them.
- **A pointer-only doc, not a real skill** — rejected: it would never be invoked on its own, which is the exact orphaned-doc failure this decision exists to avoid, and it breaks the repo's uniform skill shape.

## Consequences

- This skill is **not self-contained**: installing it without the sibling pointer edits leaves it un-routed-to. The pointer edits in `skills/esi` and `skills/sde` and the `CONTEXT-MAP.md` relationship rewrite are part of *this* skill's contract, not optional follow-up.
- The shared **SDE boundary** term now spans three contexts. A change to the boundary is a change to all three glossaries (`esi`, `sde`, and this one) — keep them consistent.
