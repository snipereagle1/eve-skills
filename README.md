# eve-skills

A monorepo of installable [agent skills](https://docs.claude.com/en/docs/claude-code/skills) for building against EVE Online's developer APIs. Each skill is a self-contained, model-invoked reference an agent consults automatically when it touches the API it covers.

## Skills

| Skill | Covers | Status |
|-------|--------|--------|
| [`esi`](./skills/esi) | Consuming the **ESI** HTTP API correctly — auth/scopes, the error budget (420/429), caching (ETag/Expires), pagination, compat-date versioning, and when to reach for the SDE instead. | Ready |
| [`sde`](./skills/sde) | Reading the **Static Data Export** — the offline dataset of static game data. Makes it legible: the domain map (~80 datasets + join keys), the build-number sync model, the JSONL `_key`/`_value` + localized-name encodings, traps, and recipes. | Ready |
| [`sde-vs-esi`](./skills/sde-vs-esi) | Deciding **which source** answers a question — ESI, the SDE, or both — and how to **join** them. The front door: routing is forced where data is single-sourced, advisory in the overlap (keyed on whether you've already integrated the SDE), plus ID→name reconciliation. | Ready |

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

Every skill is **model-invoked**: once installed it triggers on its own the moment an agent works against the API it covers. Nothing to type.

## Repo layout

```
.
├── CONTEXT-MAP.md              # the bounded contexts and the language they share
├── README.md
└── skills/
    ├── esi/                    # each subdirectory is one installable skill
    ├── sde/
    └── sde-vs-esi/
```

Every skill follows the same shape — the files below are the anatomy, not an exhaustive listing (a skill adds reference files as its domain needs them):

```
<skill>/
├── SKILL.md            # the discipline (the only file loaded at runtime)
├── manifest.yaml       # every canonical URL + volatile fact (source of truth)
├── CONTEXT.md          # the skill's glossary (its bounded context)
├── references/         # traps, recipes, domain maps — loaded on demand, never at runtime
└── docs/
    ├── adr/            # decisions specific to this skill
    └── research-notes.md   # primary-source verification trail
```

## Keeping a skill current

ESI changes under you (CCP shipped compat-date versioning, Swagger→OpenAPI, the SDE rework, and a new rate limiter — see [`skills/esi/docs/adr/0001`](./skills/esi/docs/adr/0001-currency-via-compat-date-and-manifest.md)). The design absorbs this: `SKILL.md` states only rot-resistant principles, and every volatile fact lives in `manifest.yaml`.

Each skill has a report-only refresh command — `/refresh-esi-manifest`, `/refresh-sde-manifest`, `/refresh-sde-vs-esi-manifest` — that launches the `manifest-refresher` agent to crawl the discovery sources, check anchors against their live values, and print a drift report with ready-to-paste edits. It never writes the manifest; you apply what it proposes. To re-verify by hand, walk the `manifest.yaml`: refetch each `url`, confirm it still says what the entry claims, and bump `last_verified`.

Each skill's version anchor is its own: `esi` tracks `verified_compat_date` (ESI's accepted compat dates); `sde` tracks `build` from `latest.jsonl` when a new Static Data Export ships (see [`skills/sde/docs/adr/0001`](./skills/sde/docs/adr/0001-currency-via-build-number-and-manifest.md)); `sde-vs-esi` syncs the sibling anchors and re-checks the overlap set. Entries marked `status: legacy` are kept only for people upgrading old apps.

## Design provenance

This repo was designed decision-by-decision (reference-not-workflow, distributable, model-invoked, discipline + traps + recipes, currency-via-manifest, and a front-door skill that routes between sources before either siblings fires) and grounded against live ESI docs before a line was written. The vocabulary is in `CONTEXT-MAP.md` and each skill's `CONTEXT.md`; the load-bearing decisions are in each skill's `docs/adr/`.
