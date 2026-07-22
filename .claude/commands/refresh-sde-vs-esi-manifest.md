---
description: Report-only refresh of the sde-vs-esi skill manifest (sibling-anchor sync + overlap-set drift; proposes edits, applies nothing)
---

Launch the `manifest-refresher` subagent (Task tool, `subagent_type: manifest-refresher`) against:

**Target:** `skills/sde-vs-esi/manifest.yaml`

Pass that path to the subagent and let it run its standard refresh procedure.
This manifest is **composing**: it dual-anchors by *mirroring* its siblings
(`role: mirror`) rather than checking live, and it declares an `overlap_set:` the
agent verifies against the ESI changelog + the SDE skill's dataset map. When the
subagent returns, **relay its drift report verbatim** — including every proposed
edit — then stop. Do **not** apply any edits yourself and do not edit the
manifest; the report is for me to act on.
