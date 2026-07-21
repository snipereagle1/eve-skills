# eve-skills

A monorepo of installable [agent skills](https://docs.claude.com/en/docs/claude-code/skills) for building against EVE Online's developer APIs. Each skill is a self-contained, model-invoked reference an agent consults automatically when it touches the API it covers.

## Skills

| Skill | Covers | Status |
|-------|--------|--------|
| [`esi`](./skills/esi) | Consuming the **ESI** HTTP API correctly — auth/scopes, the error budget (420/429), caching (ETag/Expires), pagination, compat-date versioning, and when to reach for the SDE instead. | Ready |
| `sde` | The **Static Data Export** — the offline dataset of static game data. | Planned |

## Installing a skill

Each skill lives in its own directory under `skills/`, so installing one is a single copy (or symlink) of that directory into a skills path:

```bash
# personal, all projects
cp -r skills/esi ~/.claude/skills/esi

# or scoped to one repo
cp -r skills/esi /path/to/repo/.claude/skills/esi

# or keep it in place and symlink
ln -s "$(pwd)/skills/esi" ~/.claude/skills/esi
```

Only the skill directory ships. The glossary, ADRs, and research notes inside it are documentation for maintainers — `SKILL.md` never loads them at runtime, so they add zero context cost when the skill fires.

The `esi` skill is **model-invoked**: once installed it triggers on its own the moment an agent works against ESI. Nothing to type.

## Repo layout

```
.
├── CONTEXT-MAP.md              # the bounded contexts and the language they share
├── README.md
└── skills/
    └── esi/                    # ← the installable skill
        ├── SKILL.md            # the discipline (the only file loaded at runtime)
        ├── manifest.yaml       # every canonical URL + volatile fact (source of truth)
        ├── CONTEXT.md          # ESI glossary (bounded context)
        ├── references/
        │   ├── traps.md        # spec-lies-to-you pitfalls, loaded on demand
        │   └── recipes.md      # multi-step task walkthroughs, loaded on demand
        └── docs/
            ├── adr/            # decisions specific to this skill
            └── research-notes.md   # primary-source verification trail
```

## Keeping a skill current

ESI changes under you (CCP shipped compat-date versioning, Swagger→OpenAPI, the SDE rework, and a new rate limiter — see [`skills/esi/docs/adr/0001`](./skills/esi/docs/adr/0001-currency-via-compat-date-and-manifest.md)). The design absorbs this: `SKILL.md` states only rot-resistant principles, and every volatile fact lives in `manifest.yaml`.

To re-verify a skill, walk its `manifest.yaml`: refetch each `url`, confirm it still says what the entry claims, and bump `last_verified` (and `verified_compat_date` if ESI's accepted compat dates changed). Entries marked `status: legacy` are kept only for people upgrading old apps.

## Design provenance

This repo was designed decision-by-decision (reference-not-workflow, distributable, model-invoked, discipline + traps + recipes, currency-via-manifest) and grounded against live ESI docs before a line was written. The vocabulary is in `CONTEXT-MAP.md` and each skill's `CONTEXT.md`; the load-bearing decisions are in each skill's `docs/adr/`.
