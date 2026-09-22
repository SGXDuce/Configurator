# Configurator JSON Export Schema — v1 design

Status: design only, nothing built yet. This is the spec to hand to Claude Code, not a description of existing behaviour.

Supersedes the open question in the project summary's §6.7 item 1 — the export shape has not been designed until now.

## Core principles this schema follows (already decided, not re-litigated)

- Flat pane list per elevation, not the internal split tree. The tree (split/sashSplit nodes, assemblyPresetRef, etc.) is editing-session bookkeeping — nobody downstream needs to know how a pane was built, only its final geometry and type.
- Configurator's own type vocabulary, not pre-mapped to any consuming tool's buckets (hinged, horizontal-slider, etc. stay as-is). Each consumer — AS 1288 now, pricing later — does its own translation. This matches §6.1's existing rule: derivation logic runs on the receiving end, never baked into the configurator itself.
- productClass is exported exactly as stored ('door' or 'window' — 'side-panel' is never set by the configurator, per §6.3/§6.4, and is not expected in this export). The sidelight/side-panel determination itself is the receiving tool's job, using the position data this schema provides.
- Position is measured from each elevation's own main origin — the bottom-left corner of the outer frame (not the opening), same point already used on-screen for dimensioning. X increases rightward, Y increases upward (opposite of the app's internal top-down drawing coordinates — this is a real conversion the export code has to do, not a relabel). Each pane's position is its own bottom-left corner — this makes the Y value directly usable as "height above floor" once the receiving tool adds its own FFL offset, with no separate "which edge" decision needed on their end.
- Always the true main origin, never a custom origin the person may have placed on screen for their own convenience while drawing (customOrigin is a display aid, not this export's reference point).
- Unframed-edge count draws from three separate sources, not one:
  - The pane's own sash*Locked flags (Case 2 — flat silicone joint to a neighbouring pane, batches 23–24).
  - Whether the pane's outer-touching edge coincides with a meeting-tagged jamb on the elevation (Case 1 — angled join, batch 26). This does not set sash*Locked on the pane at all — it's a separate mechanism, geometrically checked the same way leavesTouchingJamb() already does for the angled-join blocking rule. Missing this source would silently undercount every Case-1 pane.
  - **(Batch 30) A `type:'fixed'` pane's outer-touching edge on an elevation with `hasFrame:false`** (Case 3) — that edge has no jamb (frame is off) and no sash (fixed has none), so it's genuinely unframed even though it's neither a silicone joint nor a meeting-tagged edge. Guarded per-side against double-counting an edge that's ALSO meeting-tagged (Case 1 already counts that one). Only fires for `fixed` — every other type has its own real sash regardless of `hasFrame`. Export-only: the configurator's own live editing still allows `hasFrame:false` freely with no on-screen warning; this only affects what gets reported downstream.
- Frame length is a real sum, not a per-pane figure alone. Three components, added together (see "Frame length calculation" below) — AS 1288 doesn't need this field at all, but pricing will, so it's computed now while the geometry is fresh, per your own call.
- Case 1 fields not yet built (stored angle, cross-elevation pane link) are left out of this version entirely — not stubbed as null. When the angled-join creation flow is extended to prompt for the angle (separate future batch, per your own decision), schemaVersion bumps and those fields get added then. No placeholder debt in the meantime.

## Schema

```json
{
  "schemaVersion": 1,
  "system": {
    "elevationCount": 1,
    "totalFrameLengthMM": 3850,
    "elevations": [
      {
        "name": "Elevation 1",
        "overallWidthMM": 1200,
        "overallHeightMM": 900,
        "hasFrame": true,
        "frameMembersMM": { "head": 60, "sill": 60, "jambL": 60, "jambR": 60 },
        "meetingEdges": { "head": null, "sill": null, "jambL": null, "jambR": null },
        "frameLengthMM": 3850,
        "panes": [
          {
            "id": "F",
            "productClass": "window",
            "type": "fixed",
            "hingeEdge": null,
            "slideDirection": null,
            "bladeWidthMM": null,
            "bladeLengthMM": null,
            "xMM": 0,
            "yMM": 0,
            "widthMM": 600,
            "heightMM": 780,
            "areaM2": 0.47,
            "visibleGlazedAreaM2": 0.47,
            "sashEdgesMM": { "top": 0, "bottom": 0, "left": 0, "right": 0 },
            "unframedEdgeCount": 0
          },
          {
            "id": "H",
            "productClass": "window",
            "type": "hinged",
            "hingeEdge": "left",
            "slideDirection": null,
            "bladeWidthMM": null,
            "bladeLengthMM": null,
            "xMM": 600,
            "yMM": 0,
            "widthMM": 600,
            "heightMM": 780,
            "areaM2": 0.47,
            "visibleGlazedAreaM2": 0.31,
            "sashEdgesMM": { "top": 40, "bottom": 40, "left": 40, "right": 40 },
            "unframedEdgeCount": 0
          }
        ]
      }
    ]
  },
  "rawState": { "...": "see \"rawState — raw internal state block\" below; opaque to any consumer" }
}
```

A 2-elevation (angled join) system exports both elevations under `system.elevations[]`, each with its own `meetingEdges` populated where relevant. `totalFrameLengthMM` at the system level sums both elevations' own `frameLengthMM`.

## Field notes

| Field | Notes |
|---|---|
| `productClass` | `'door'` or `'window'` as stored. Side-panel/sidelight classification is NOT done here — receiving tool derives it from position + door adjacency (§6.3), unchanged from the existing architecture decision. |
| `type` | Raw configurator vocabulary (`fixed`, `fixed-framed`, `hinged`, `horizontal-slider`, `vertical-slider`, `louvre`). No bucket mapping applied. |
| `xMM` / `yMM` | Pane's own bottom-left corner, from the elevation's true main origin, Y increasing upward. Converted from the app's internal top-down `walk()` coordinates — this is real conversion logic, not a passthrough. |
| `sashEdgesMM` | Raw per-edge widths as stored. `fixed` panes always report all-zero (matches existing `glazedAreaM2` special-case). |
| `unframedEdgeCount` | Combines `sash*Locked` flags (Case 2), meeting-jamb-touching geometry (Case 1), and (batch 30) a `fixed` pane's outer-touching edge on a `hasFrame:false` elevation (Case 3) — see above. |
| `visibleGlazedAreaM2` | Uses the renamed label from batch 28 — same computation as `areaM2` minus sash, unchanged. |
| `frameLengthMM` | See calculation below. Not needed by AS 1288 — included for the future pricing tool. |

## Frame length calculation

Three components summed per elevation:

1. Each pane's own sash perimeter — `2 × (widthMM + heightMM)` for any pane with a real sash (i.e. not `fixed`). A `fixed` pane contributes 0 here.
2. Real mullions/transoms between panes — walk the split tree once; for every `split` node with `thicknessMM > 0` and NOT `isSiliconeJoint`, add its own length (a vertical mullion's length is the height of the region it divides; a horizontal transom's length is the width).
3. Outer frame — head/sill add the elevation's overall width, jambL/jambR add the overall height, UNLESS that member is off (`hasFrame: false`) or meeting-tagged, in which case it contributes 0.

Do NOT add anything extra for double-hung or sliding-overlap joins. Those panels are already sized wider to include their overlap material — it's already counted in component 1. There is no real separate transom/mullion at an overlap join (confirmed in the project doc itself — the top and bottom sash rails of a double-hung genuinely interlock with nothing between them). Adding a component-2 entry for these would double count.

Known simplification, flagged not hidden: each frame member's length is its full span (e.g. a 1800mm-wide head counts as 1800mm), not the shorter length after mitring the corners. Real corners lose a small amount to the mitre cut, unmodelled here. Accepted for now per your own call — revisit when building the pricing tool properly.

## `rawState` — raw internal state block (batch 37)

A second, separate top-level field, sitting alongside `system`, not inside it. Where `system` is a deliberately simplified, derived flat pane list (§'s own core principle above — the tree is "editing-session bookkeeping" that "nobody downstream needs to know"), `rawState` is the opposite: the configurator's actual live internal state, sufficient to fully reconstruct an editing session later (preset refs, hinged-door refs, sash locks, meeting edges, the split tree itself — everything `system` deliberately discards).

**`rawState` is explicitly opaque to any consumer.** The AS 1288 tool (or any other reader) stores and returns it unread — it never parses or derives anything from `rawState`'s own contents. This is why `schemaVersion` does not cover it: `schemaVersion` versions `system`'s own derived shape only. `rawState`'s own internal shape can change freely between configurator versions without being a breaking change for any consumer, precisely because no consumer is expected to read it — it only round-trips through whatever external system stores it, until the configurator itself (via `restoreFromRawState()`) reads it back.

Produced by `serializeRawState()`, consumed by `restoreFromRawState()` — both in `configurator_prototype_oxxo_8.html`. Shape, as of batch 38:

```json
{
  "rawStateSchemaVersion": 1,
  "elevations": [ /* the full live elevation objects — tree, overallW/overallH, hasFrame,
                     frameMembers, meetingEdges, meetingLink, selected, pendingSplit, etc. —
                     not a derived form, the actual internal representation */ ],
  "activeElevationIndex": 0,
  "angledJoinAngleDeg": null,
  "angledJoinType": null
}
```

**`rawStateSchemaVersion` (added batch 38).** Batch 37 deliberately left `rawState` unversioned, since it's opaque to the AS 1288 tool and nothing external was expected to read it. Batch 38's embed-mode "load" handshake changed that: the CONFIGURATOR ITSELF now needs to recognize its own state shape before trusting a `rawState` blob handed back to it on load — so a minimal version marker was added specifically to give that check something real to validate against, rather than skipping it. This is a small, deliberate scope addition beyond what batch 37 shipped, flagged here rather than folded in silently. It has no relationship to `schemaVersion` (which still only describes `system`'s own derived shape) — bump `rawStateSchemaVersion` only when `restoreFromRawState()`'s expectations of what `serializeRawState()` produces change in an incompatible way.

`restoreFromRawState()` is now wired into the embed-mode "load" message (see "postMessage embed protocol" below) — no longer prerequisite-only as of batch 38.

## postMessage embed protocol (batch 38)

When this configurator is opened inside an `<iframe>` (`window.parent !== window`), it runs a same-origin postMessage handshake with the parent page instead of behaving as a standalone tool. **Standalone mode (opened directly, not embedded) is completely unaffected** — no listener is ever registered, and "Export JSON" keeps downloading a file exactly as before.

**Security — same-origin only, no hardcoded URL:**
- Every message this configurator SENDS targets `window.parent` with `window.location.origin` as the explicit target origin — never `'*'`.
- Every message this configurator RECEIVES is checked against BOTH `event.origin === window.location.origin` AND `event.source === window.parent` before anything in it is read. Failing either check is a silent drop — no error, no console output, no partial processing.

**Handshake, in order:**

1. **`ready`** — sent by the configurator once its own initial render has genuinely finished (not before): `{ "type": "ready", "schemaVersion": 1 }`. `schemaVersion` here is the export's own `system` schema version (currently 1) — this tells the parent which derived-pane-list shape a subsequent `done` will use.
2. **`load`** — sent by the parent in response, exactly once: `{ "type": "load", "state": null | <a previously-saved rawState blob> }`.
   - `state: null` — start a fresh, single-elevation session (the existing default). `restoreFromRawState()` is never called.
   - `state: { ...a real rawState object... }` — its own `rawStateSchemaVersion` is checked first. If it doesn't match the configurator's own `RAW_STATE_SCHEMA_VERSION`, the configurator shows a persistent, visible in-page error (not a console log, not an `alert()`) and stops — `restoreFromRawState()` is never called, no partial or best-effort load is attempted. If it matches, `restoreFromRawState(state)` runs and the session is fully reconstructed.
3. The user edits normally — nothing about the editing UI differs in embed mode.
4. **`done`** — sent by the configurator when the user finishes: `{ "type": "done", "export": <the full buildExportData() output, i.e. { schemaVersion, system, rawState }> }`. This is the ONLY way `done` ever fires. It's driven by the same "Export JSON" button, relabelled "Done" in embed mode — one button, one conditional action, rather than a second embed-only control. In embed mode this button sends the postMessage instead of downloading a file.

**No cancel/close from the configurator.** Per the agreed protocol, cancel/close belongs entirely to the parent page's own modal chrome (its own close button). The configurator renders no close/cancel control in any mode and never sends a cancel-type message — only `ready` and `done` are ever constructed.

**No row/system ID handling.** The parent tracks which schedule row is open; the configurator neither accepts, stores, nor echoes back any kind of identifier.

## What's deliberately NOT in this version

- Cross-elevation pane-to-pane link (§7.4) — concrete shape still undesigned, separate from this batch. Once built, `system` will need a schema bump (v2) to represent it in the derived pane list.
- FFL and room type — confirmed to live entirely on the AS 1288 tool's own schedule-row form, never in this export (§6.5, §6.7).
- Any AS 1288-specific derivation (sidelight test, glazing-method bucketing) — runs on the receiving end, not here.
- The angled join's stored angle (`angledJoinAngleDeg`, batch 35) and construction type (`angledJoinType`, batch 36) are both now built in-app, but deliberately still NOT part of `system`'s own derived shape — they live only in `rawState` (opaque) for now. Exposing them as real, documented `system`-level fields is exactly the v2 work the cross-elevation-pane-link bump above is already being held for; they won't be released into `system` piecemeal ahead of it.
- Any row/system identifier — the parent owns this entirely; the postMessage protocol (batch 38) neither accepts nor returns one.
