# ESI Skill

A distributable, model-invoked reference skill that supplies agents the cross-cutting discipline for consuming EVE Online's ESI API correctly. It is a knowledge artifact, not a procedure, and never duplicates the live OpenAPI spec.

## Language

**Discipline**:
The always-loaded spine of the skill — the rot-resistant, HTTP-level best practices an agent needs before touching ESI (error-limit budget, caching/ETags, pagination, scopes, the SDE boundary).
_Avoid_: guidelines, rules, tips

**Trap**:
A non-obvious ESI pitfall where the spec under-documents or misleads, costing hours (e.g. Ansiblex destination encoded only in the structure name string). Disclosed in a separate reference file, never in the spine.
_Avoid_: gotcha, bug, edge case

**Recipe**:
A short walkthrough for a confusing multi-step ESI task that names the specific endpoints and scopes to use. Disclosed, not in the spine. Points at the spec for endpoint shapes.
_Avoid_: workflow, tutorial, guide

**Documentation manifest**:
The structured `manifest.yaml` enumerating every canonical doc/URL the skill references, each with `url`, `purpose`, `kind`, and `last_verified`/`compat_date`. The single place to update when ESI changes; walkable for drift-checking.
_Avoid_: sources list, links, bibliography

**Compat date**:
ESI's compatibility-date versioning mechanism, used as the skill's version anchor — the skill pins what it verified against a compat date rather than to volatile route versions.
_Avoid_: version, api version, route version

**SDE boundary**:
The first-class best practice of choosing datasource by data kind — static/reference data (type names, topology, dogma) belongs to the Static Data Export; dynamic/character/live data belongs to ESI. The skill owns the boundary, not the SDE's contents.
_Avoid_: SDE integration, static data handling
