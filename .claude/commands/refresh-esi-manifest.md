---
description: Report-only refresh of the ESI skill manifest (discovery + drift; proposes edits, applies nothing)
---

Launch the `manifest-refresher` subagent (Task tool, `subagent_type: manifest-refresher`) against:

**Target:** `skills/esi/manifest.yaml`

Pass that path to the subagent and let it run its standard refresh procedure.
When it returns, **relay its drift report verbatim** — including every proposed
edit — then stop. Do **not** apply any edits yourself and do not edit the
manifest; the report is for me to act on.
