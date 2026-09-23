# Configurator JSON Export Schema — v4 (v1 superseded batch 39, v2 superseded batch 43, v3 superseded batch 44)

Status: built (v1 in batch 29, v2/v3 in batches 39/43, v4 in batch 44, corrected batch 45) — this document now describes the actual, current export shape, not a design-only proposal.

Originally written to resolve the open question in the project summary's §6.7 item 1 — the export shape had not been designed before v1.

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
- **(v2, batch 39; matching rule corrected batch 40; tolerance corrected batch 41)** The cross-elevation pane link is NOT a stored field anywhere — no new tree-node field, no UI to create/manage/edit it. It's computed fresh, directly from geometry, every single time an export is produced, so there's nothing to go stale. Each side's meeting-tagged jamb is found independently (never assumed to be jambL on one elevation and jambR on the other). Panes are matched by their actual edge position along the jamb (both start and end, within `PANE_EDGE_MATCH_TOL_MM`, 1mm) — NOT by sorted order and array index alone, which batch 39 originally did and batch 40 found to be a real bug (two elevations with the same pane count in the same order can still have misaligned edges). Any pane with no matching partner blocks the ENTIRE export — see "Export blocked on an unmatched meeting-jamb pane" below.
- **(v2, batch 39)** `angledJoinAngleDeg`/`angledJoinType` (batches 35/36) were deliberately held out of `system` until this pane link could ship alongside them, per this doc's own earlier note — both are now real, documented `system`-level fields, not stubs.
- **(v4, batch 44)** `unframedEdgeCount` sums three sources without saying which specific edges they are — AS 1288 needs the specific edges (Table 5.3 for vertical/jamb-side edges vs. a different clause for horizontal edges), not just a count. `unframedEdges: {top, bottom, left, right}` is the same three-source computation as `unframedEdgeCount`, mapped per-edge instead of summed — see the `unframedEdges` field note below for the two places the two fields deliberately disagree.
- **(batch 45, no schemaVersion bump — a correctness fix to the existing v4 shape, not a new field)** Two corrections, both scoped to sashless sliding-window presets only. First, `buildAssemblyPresetTree` (the underlying preset builder, not this export layer) had wrongly given a sashless `O` (`type:'fixed'`) leaf `SASHLESS_STILE_MM` on its left/right sash edges — a bare `fixed` pane has no sash material at all, so it's now zero on all four edges, matching an ordinary non-sashless `O`. Second, a new Case 4 on `unframedEdges` (only — `unframedEdgeCount` is deliberately untouched) flags a sashless `O`'s edge at an internal overlap join (touching a sashless `X` sibling) as unframed, since that edge now genuinely has no sash. See the `unframedEdges` field note below for the full boundary.

## Schema

```json
{
  "schemaVersion": 4,
  "system": {
    "elevationCount": 2,
    "totalFrameLengthMM": 7100,
    "angledJoinAngleDeg": 135,
    "angledJoinType": "mitred",
    "elevations": [
      {
        "name": "Elevation 1",
        "overallWidthMM": 1200,
        "overallHeightMM": 900,
        "hasFrame": true,
        "frameMembersMM": { "head": 60, "sill": 60, "jambL": 60, "jambR": 60 },
        "meetingEdges": { "head": null, "sill": null, "jambL": null, "jambR": "meeting" },
        "frameLengthMM": 3550,
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
            "unframedEdgeCount": 0,
            "unframedEdges": { "top": false, "bottom": false, "left": false, "right": false },
            "linkedPaneId": null,
            "sashless": false
          },
          {
            "id": "F2",
            "productClass": "window",
            "type": "horizontal-slider",
            "hingeEdge": null,
            "slideDirection": "right",
            "bladeWidthMM": null,
            "bladeLengthMM": null,
            "xMM": 600,
            "yMM": 0,
            "widthMM": 660,
            "heightMM": 900,
            "areaM2": 0.59,
            "visibleGlazedAreaM2": 0.57,
            "sashEdgesMM": { "top": 0, "bottom": 0, "left": 10, "right": 10 },
            "unframedEdgeCount": 1,
            "unframedEdges": { "top": false, "bottom": false, "left": true, "right": false },
            "linkedPaneId": "F",
            "sashless": true
          }
        ]
      },
      {
        "name": "Elevation 2",
        "overallWidthMM": 1000,
        "overallHeightMM": 900,
        "hasFrame": true,
        "frameMembersMM": { "head": 60, "sill": 60, "jambL": 60, "jambR": 60 },
        "meetingEdges": { "head": null, "sill": null, "jambL": "meeting", "jambR": null },
        "frameLengthMM": 3550,
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
            "widthMM": 500,
            "heightMM": 900,
            "areaM2": 0.45,
            "visibleGlazedAreaM2": 0.45,
            "sashEdgesMM": { "top": 0, "bottom": 0, "left": 0, "right": 0 },
            "unframedEdgeCount": 1,
            "unframedEdges": { "top": false, "bottom": false, "left": true, "right": false },
            "linkedPaneId": "F2",
            "sashless": false
          }
        ]
      }
    ]
  },
  "rawState": { "...": "see \"rawState — raw internal state block\" below; opaque to any consumer" }
}
```

A 2-elevation (angled join) system exports both elevations under `system.elevations[]`, each with its own `meetingEdges` populated where relevant. `totalFrameLengthMM` at the system level sums both elevations' own `frameLengthMM`. `angledJoinAngleDeg`/`angledJoinType` are `null` on a single-elevation system, and every pane's `linkedPaneId` is `null` throughout in that case too — there's no meeting jamb to check. (The example's pane `"F2"` is shown as a sashless sliding-window sash purely to illustrate the field — real geometry for an actual sashless `OX-win` preset would also differ in exact widths per the batch-42 tuck-in formula; not meant as a literal worked example of that calculation.)

Note in the example above: `id` values are scoped per elevation (each elevation labels its own panes independently, starting fresh), so `linkedPaneId` is only meaningful together with knowing which elevation it refers to — Elevation 1's pane `"F2"` (`linkedPaneId: "F"`) refers to Elevation 2's pane `"F"`, not another pane within Elevation 1 itself. With a hard cap of 2 elevations, "the other elevation" is always unambiguous — no elevation index is included in `linkedPaneId`.

## Field notes

| Field | Notes |
|---|---|
| `productClass` | `'door'` or `'window'` as stored. Side-panel/sidelight classification is NOT done here — receiving tool derives it from position + door adjacency (§6.3), unchanged from the existing architecture decision. |
| `type` | Raw configurator vocabulary (`fixed`, `fixed-framed`, `hinged`, `horizontal-slider`, `vertical-slider`, `louvre`). No bucket mapping applied. |
| `xMM` / `yMM` | Pane's own bottom-left corner, from the elevation's true main origin, Y increasing upward. Converted from the app's internal top-down `walk()` coordinates — this is real conversion logic, not a passthrough. |
| `sashEdgesMM` | Raw per-edge widths as stored. `fixed` panes always report all-zero (matches existing `glazedAreaM2` special-case). |
| `unframedEdgeCount` | Combines `sash*Locked` flags (Case 2), meeting-jamb-touching geometry (Case 1), and (batch 30) a `fixed` pane's outer-touching edge on a `hasFrame:false` elevation (Case 3) — see above. **Unchanged by batch 44 or batch 45** — still the plain sum of the three sources above only, including for a sashless pane and including a sashless `O` at an internal overlap join. See `unframedEdges` below for the two distinct, independently-documented cases where this diverges from it. |
| `unframedEdges` (v4; Case 4 added batch 45) | `{top, bottom, left, right}` booleans — the same three sources as `unframedEdgeCount` (Cases 1–3), mapped per-edge instead of summed, so the receiving tool knows which specific edges are unframed (needed to route AS 1288 Table 5.3 for vertical/jamb-side edges vs. a different clause for horizontal edges — `unframedEdgeCount` alone can't say which). Two distinct exceptions apply on top of Cases 1–3, each documented separately — do not conflate them: **Sashless top/bottom exception (batch 44):** if the pane is sashless (batch 42), `top` and `bottom` are always forced `false` here regardless of what the three-source computation produced — a sashless pane's rail-free horizontal edges are reported only via `sashless: true`, never via `unframedEdges.top`/`.bottom`. `left`/`right` are untouched by this exception. **Case 4 — sashless internal overlap join (batch 45):** a sashless `O` (`type:'fixed'`) leaf's edge at an internal sliding-window overlap join (i.e. touching a sashless `X` sibling, not an outer jamb/head/sill boundary) is also unframed — as of batch 45, that edge genuinely carries zero real sash material (batch 45 also fixed `buildAssemblyPresetTree` itself so a sashless `O`'s own `sashLeftMM`/`sashRightMM` are `0`, matching an ordinary bare fixed pane; batch 42 had wrongly stamped `SASHLESS_STILE_MM` there). Gated strictly on the pane's own `sashless === true && type === 'fixed'` — never on a neighbour's value, and never applied to an ordinary (non-sashless) `O` next to an ordinary `X`, which is genuinely framed by real sash material there. Both exceptions mean `unframedEdges`'s own true-count can legitimately be LOWER than `unframedEdgeCount` for a sashless preset — most visibly for a sashless `'O'` panel that is both at an internal overlap join AND on a `hasFrame:false` elevation, where `unframedEdgeCount` still counts every edge from Cases 1–3 unchanged while `unframedEdges` reflects both exceptions. Deliberate and documented in both cases, not a bug. |
| `visibleGlazedAreaM2` | Uses the renamed label from batch 28 — same computation as `areaM2` minus sash, unchanged. |
| `frameLengthMM` | See calculation below. Not needed by AS 1288 — included for the future pricing tool. |
| `linkedPaneId` (v2) | The other elevation's pane `id` this pane is joined to along the angled join's meeting jamb, or `null` if this pane doesn't touch that jamb (or no join exists at all). Computed fresh every export — never stored, never editable. See "Cross-elevation pane link" below. |
| `angledJoinAngleDeg` / `angledJoinType` (v2, system-level) | The join's stored angle (degrees, 90–180) and construction type (`'butt'` or `'mitred'`) — both `null` on a single-elevation system. Live module-level values as captured at "Add angled join" time (batches 35/36), editable afterward via the persistent controls next to "Remove angled join". |
| `sashless` (v3) | `true` if this pane was built by a sashless sliding-window or double-hung preset (batch 42 — near-zero top/bottom sash, a narrow fixed stile left/right), `false` for every other pane. A direct, undecorated read of the pane's own internal `sashless` flag — no derivation, no inference from `sashEdgesMM`'s actual values. `false` (never omitted) for every pane that isn't sashless, including every pane that existed before batch 42. |

## Cross-elevation pane link (v2, batch 39; matching rule corrected batch 40)

Computed fresh at export time, directly from geometry — **not a stored field anywhere in the app's own data model.** There is no new field on any tree node and no UI to create, edit, or remove a link; nothing about it can go stale, because it's recomputed from scratch on every single export.

For each elevation, the algorithm finds whichever of its four `meetingEdges` is actually tagged `'meeting'` (checked independently per elevation — never assumed to be `jambL` on one side and `jambR` on the other, since `addAngledJoin` always tags the two elevations' OPPOSITE jambs, not a fixed pair) and collects every leaf touching that edge. Every leaf touching a meeting-tagged jamb is already guaranteed `type:'fixed', productClass:'window'` by the existing join-creation/edit blocking rule (`leafAllowedAgainstMeetingEdge`), so no type restriction was needed to make matching meaningful.

**Matching is by actual edge position, not by sorted order and array index.** This was found to be a real correctness bug in the original v2 batch (39) and fixed in batch 40, a configurator-side correctness finding — not a change requested by the AS 1288 side. The original approach sorted each elevation's touching panes top-to-bottom and paired them purely by index (first-with-first, second-with-second): that's wrong whenever the two elevations have the same pane count in the same top-to-bottom order but the panes' actual edges along the jamb don't line up — e.g. a transom at 1000mm on one elevation's meeting edge and a transom at 1400mm on the other's, with a single pane on each side either side of it. Index-pairing would link panes that don't physically meet at all.

The corrected rule: each touching pane's start and end position along the jamb (its own internal top-down `y` and `y + h` — the same coordinate already used to detect that it's touching the jamb in the first place, not a second coordinate system) must both match — within `PANE_EDGE_MATCH_TOL_MM` — a pane on the other elevation before the two are linked. A plain pane-count mismatch is just one way for a pane to end up with no matching partner under this rule, so it's no longer a separate check — there is one match check per touching pane.

**`PANE_EDGE_MATCH_TOL_MM` (1mm, corrected batch 41) is deliberately a SEPARATE constant from `EDGE_TOUCH_EPS_MM` (0.01mm), not a reuse of it.** Batch 40 originally reused `EDGE_TOUCH_EPS_MM` for this comparison — found to be wrong. `EDGE_TOUCH_EPS_MM` was built for a different, more forgiving question ("does this pane touch the jamb at all," computed from clean subtraction/addition through one elevation's own tree, where 0.01mm exists only to absorb floating-point noise), not "do two independently-computed mm values, from two different elevations' own trees, represent the same physical edge" — where 0.01mm is unrealistically tight for physical timber joinery and risked false-blocking a genuinely valid export over ordinary float drift. `PANE_EDGE_MATCH_TOL_MM` is used only in this one edge-matching comparison; `EDGE_TOUCH_EPS_MM` itself, and every other place it's used (jamb-touching detection, `unframedEdgeCount`), is unchanged.

The common case is exactly one pane per side, which pairs trivially provided their edges do actually align (usually true, since the join creation flow syncs overall height — but not guaranteed, since each elevation's own frame members can still differ). Multiple stacked panes against the meeting jamb on both sides pair by matching height range, not merely by stacking order.

## Export blocked on an unmatched meeting-jamb pane (v2, batch 39; scope widened batch 40)

If any pane touching either elevation's meeting jamb has no matching partner on the other side — whether because the counts differ, or because the counts match but the edges don't line up — **the entire export is blocked**: neither the standalone file download nor the embed `done` message is produced. A clear, persistent, visible in-page error is shown (never a console log or `alert()`), naming the elevation and the specific height range (in mm) that has no match, and the closest candidate found on the other side if any panes touch that jamb at all. As of batch 40 this error shows in two places: the existing top-of-page banner, and a second box directly next to the Export/Done button itself, so the error is visible both from a distance and right at the point of the action that triggered it.

This check runs first, before any part of the export (including `rawState`) is built — a mismatch produces genuinely no export, not a partial or malformed one — and is identical in standalone and embed mode; it isn't an embed-only concern.

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

1. **`ready`** — sent by the configurator once its own initial render has genuinely finished (not before): `{ "type": "ready", "schemaVersion": 4 }`. `schemaVersion` here is the export's own `system` schema version (v1 → v2 in batch 39, v2 → v3 in batch 43, v3 → v4 in batch 44) — this tells the parent which derived-pane-list shape a subsequent `done` will use, and it always matches `done`'s own `export.schemaVersion`.
2. **`load`** — sent by the parent in response, exactly once: `{ "type": "load", "state": null | <a previously-saved rawState blob> }`.
   - `state: null` — start a fresh, single-elevation session (the existing default). `restoreFromRawState()` is never called.
   - `state: { ...a real rawState object... }` — its own `rawStateSchemaVersion` is checked first. If it doesn't match the configurator's own `RAW_STATE_SCHEMA_VERSION`, the configurator shows a persistent, visible in-page error (not a console log, not an `alert()`) and stops — `restoreFromRawState()` is never called, no partial or best-effort load is attempted. If it matches, `restoreFromRawState(state)` runs and the session is fully reconstructed.
3. The user edits normally — nothing about the editing UI differs in embed mode.
4. **`done`** — sent by the configurator when the user finishes: `{ "type": "done", "export": <the full buildExportData() output, i.e. { schemaVersion, system, rawState }> }`. This is the ONLY way `done` ever fires. It's driven by the same "Export JSON" button, relabelled "Done" in embed mode — one button, one conditional action, rather than a second embed-only control. In embed mode this button sends the postMessage instead of downloading a file.

**No cancel/close from the configurator.** Per the agreed protocol, cancel/close belongs entirely to the parent page's own modal chrome (its own close button). The configurator renders no close/cancel control in any mode and never sends a cancel-type message — only `ready` and `done` are ever constructed.

**No row/system ID handling.** The parent tracks which schedule row is open; the configurator neither accepts, stores, nor echoes back any kind of identifier.

## What's deliberately NOT in this version

- FFL and room type — confirmed to live entirely on the AS 1288 tool's own schedule-row form, never in this export (§6.5, §6.7).
- Any AS 1288-specific derivation (sidelight test, glazing-method bucketing) — runs on the receiving end, not here.
- Any row/system identifier — the parent owns this entirely; the postMessage protocol (batch 38) neither accepts nor returns one.

**Now built, as of v2 (batch 39) — no longer open:** the cross-elevation pane-to-pane link (computed fresh at export time, not stored — see "Cross-elevation pane link" above), and `angledJoinAngleDeg`/`angledJoinType` as real, documented `system`-level fields (previously held back in batches 35–37, deliberately, until this pane link could ship alongside them).
