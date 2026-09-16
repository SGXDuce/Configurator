# Window/Door Elevation Configurator — Project Summary
### Version: 2.1 (supersedes v1.0 — covers batches 1 through 16, plus AS 1288 integration planning and the depth/glass data model)
### Last updated: 16 September 2026

---

## 0. How to use this document

Paste this into a new chat under this project along with the current `configurator_prototype_oxxo_8.html`, and say: "I am continuing work on the standalone window/door elevation configurator. This document summarises everything decided and built so far. We are now working on: [describe the new task]."

This is the authoritative record of what exists and what's been decided. Don't re-derive or silently override anything in here — if something looks wrong, say so and ask before changing direction. This version was written from chat history, not from re-reading the live HTML file line by line — if anything here contradicts what the actual file does, the file is the source of truth for code behaviour; flag the discrepancy rather than trusting this doc blindly.

## 1. What this is and why it exists

A standalone, browser-based configurator for building up a window/door elevation: an opening, an optional outer frame, and a recursively-divided grid of panes, each assignable a type and a door/window classification. Every dimension is shown directly on the drawing and is editable there.

It grew out of, but stays deliberately separate from, the AS 1288 Glass Thickness Calculator project at Duce Timber Windows and Doors. It's explicitly not a calculator-specific feature — the intent is a standalone tool other tools can consume: the AS 1288 compliance engine (needs per-pane span, aspect ratio, framing condition), and a future pricing/quoting tool (needs per-pane glass area, frame linear lengths, counts).

Open strategic question, still unresolved, worth raising again before real investment: Duce has a Logikal vendor evaluation in progress (a commercial fenestration ERP with its own parametric configurator and quoting), and a separate parked Fusion 360 drawing-handoff initiative. Building this configurator in-house may duplicate either. This question resurfaced when discussing a genuine 3D parametric model (see §8) — the same duplication risk applies there, at a much larger scale, and was a deciding factor in choosing a data-only approach over a 3D rendering approach for depth/z-axis capture.

Current state of the actual artifact: a single self-contained HTML file (`configurator_prototype_oxxo_8.html`), vanilla JS, no frameworks or build step, no backend, no persistence. Nothing is wired into the AS 1288 Flask app yet, though the integration architecture is decided (§6) and one standalone piece of it has been built and tested (§6.4) **elsewhere** — not in this repo (see §9's note on environment gaps). Built entirely inside Claude chat sessions and Claude Code sessions, iterated batch by batch.

## 2. Data model (current state, post-batch-16)

### 2.1 — The two-level concept this sits inside (upstream context, not literally coded yet)

The broader "window schedule" concept this configurator serves is two levels: Opening (a schedule row/job line item) and System(s) (one or more glazed units within that opening). The configurator prototype works on one system/frame at a time. Multi-system placement remains parked (§5).

### 2.2 — The tree

A single rectangular region is recursively divided by a binary tree of nodes:

- `{kind:'empty'}` — an unassigned region.
- `{kind:'filled', type, productClass, sashTopMM, sashBottomMM, sashLeftMM, sashRightMM, hingeEdge?, slideDirection?, bladeWidthMM?, bladeLengthMM?}` — an assigned pane. See §2.3 for the current type vocabulary and §2.5 for the per-edge sash dimensions.
- `{kind:'split', axis, posMM, thicknessMM, children:[a,b], hingedDoorRef?, assemblyPresetRef?}` — a division into two children. `hingedDoorRef` and `assemblyPresetRef` are markers stamped at creation time so the tree can be walked back up to find "is this part of a hinged door" or "is this part of a built preset assembly" without re-deriving it from shape — shape alone is not reliable once a split node's own identity is overwritten by the split itself.
- `{kind:'sashSplit', axis, fromEdge, sashSizeMM, overlapMM, children}` — the original overlap-sash mechanism, kept, not discarded.

Splitting always divides an existing region into two. No free-form x/y placement.

### 2.3 — Pane types

Six types:

| Type | Meaning |
|---|---|
| `fixed` | No sash frame of its own — glass runs to the pane's own edge |
| `fixed-framed` (label: "Fixed (sash)") | Non-operable, but has a real sash frame band, same as any operable type. Window/side-panel only — excluded from doors |
| `hinged` | `hingeEdge: 'left'\|'right'\|'top'\|'bottom'`. Covers casement and awning as one underlying type |
| `horizontal-slider` | `slideDirection: 'left'\|'right'` |
| `vertical-slider` | `slideDirection: 'up'\|'down'` |
| `louvre` | `bladeWidthMM`, `bladeLengthMM` stored — captured for the future AS 1288 blade-envelope check, not yet wired into rendering realism |

Toolbar-level restriction, not a data-model restriction: the door group's "Hinged" button and the window group's "Casement" button both produce `type:'hinged'`, restricted to `hingeEdge: 'left'|'right'` only. A separate "Awning" button produces `type:'hinged', hingeEdge:'top'` directly, with no edge dropdown shown. `hingeEdge:'bottom'` is supported in the data model but unreachable from any button.

Two toolbar rows — "Door panels" (fixed, hinged, horizontal-slider) and "Window sashes" (fixed, fixed-framed, hinged/casement, awning, horizontal-slider, vertical-slider, louvre).

### 2.4 — productClass and nesting inside a hinged door

Every filled node carries `productClass: 'door' | 'window' | 'side-panel'`. `side-panel` is supported in the data model but not settable from any toolbar button — reserved for a future derivation rule (§6.3), not a manual choice.

Nesting a sash inside a hinged door panel: splitting a hinged door panel allows `louvre`, `vertical-slider`, and `hinged` (only when `hingeEdge:'top'`, i.e. the Awning button) as children — `horizontal-slider` and side-hung `hinged` are blocked. `fixed` is always allowed as a child and renders with no frame band of its own. Any type assigned while nested under a hinged door has its `productClass` forced to `'door'`, regardless of which toolbar group the button came from — opening type (door/window/side-panel) is independent of glazing method.

The hinge symbol for a split hinged door spans the entire original door panel, not the individual leaf — `walk()` tracks every split node's own rectangle, not just leaf rectangles, and finds the nearest hinged-door ancestor to size the symbol against. A nested Awning sash still draws its own hinge symbol in addition — only a plain `hingeEdge:left/right` leftover (the un-nested continuation of the door's own glazing) is suppressed in favour of the door-spanning symbol.

Hinge symbol rendering: two full-corner-to-opposite-midpoint SVG lines (replaced an earlier CSS border-triangle) — apex points away from the hinge edge.

### 2.5 — Per-edge sash frame dimensions and glazed area (batch 9)

Every filled pane stores four independent values — `sashTopMM`, `sashBottomMM`, `sashLeftMM`, `sashRightMM` — replacing an earlier single uniform `sashFrameMM`. The toolbar still shows one input ("Sash frame width (mm)"); typing a value writes all four fields equally, for manually created/edited panes. This was built as backend flexibility for a future per-edge UI, not yet exposed for independent editing.

**Batch 16 addition — real per-type default shape for preset-created panes:** `PRESET_SASH_EDGES = {sashTopMM:60, sashLeftMM:60, sashRightMM:60, sashBottomMM:86}`, sourced from Duce's own dimensional data (§7.1), is stamped on every casement and awning leaf built via the assembly-preset mechanism (`buildCasementPresetTree`/`buildAwningPresetTree`). This does **not** change the manual toolbar path (`makeTypeButton`), which still uses the uniform `DEFAULT_SASH_MM` fallback for hand-assigned/edited panes — the distinction is deliberate: only a *preset* stamps the real per-type shape at creation, per this batch's own instruction. Double-hung is explicitly excluded from this batch's per-edge treatment — see §2.9's double-hung note below.

A separate `glazedAreaM2(leaf)` computes true glass area (pane size minus the four sash values), shown in an additive pane-table column, "Glazed area (m²)" — the original "Area (m²)" column (full outer pane area) is unchanged.

### 2.6 — Outer frame

Head/sill/jambL/jambR, independently editable, off by default.

### 2.7 — Master dimension flip

The entered overall width/height are the outer frame's own outside dimensions. The opening is derived: `openingW = overallW − jambL − jambR`, `openingH = overallH − head − sill`. Changing a frame member cascades a recursive minimum-pane-size validation (`validateTreeAgainstSize`) against every descendant, blocking the edit inline if anything would shrink below `MIN_PANE_MM`. This validation function is reused directly by the assembly-preset reopen mechanism (§2.9).

### 2.8 — Dimensioning

AS 1100 chain dimensioning: when multiple splits along one axis would produce colliding on-screen labels, later ones stack into additional rows further from the drawing. Pane dimension labels and the door/window (D/W) selected-pane indicator both show only when that pane is currently selected, not unconditionally by on-screen size.

### 2.9 — Assembly presets (batches 10–16)

A way to generate a full multi-section horizontal layout (or, for double-hung, a two-level stacked layout) in one action, instead of manually splitting and assigning each section by hand.

**Entry point:** a categorized, icon-based picker — no typed pattern entry anywhere (an earlier free-text field was fully removed, not left as a fallback). Icons are auto-generated small SVG previews reusing the same visual language as the main drawing, via a shared family-aware dispatch mechanism (`getPresetSectionCount`/`buildPresetTree`/`renderIconForPreset`) so each family's builder/icon logic is a swappable implementation behind one dispatch point.

**Families, as currently built:**

- **Sliding doors** — OX, OXX, OXXO, XOX, OXXX, OXXXX, OXXXXX. `O` → `fixed`, `X` → `horizontal-slider`. Every section defaults `productClass:'door'`.
- **Sliding windows** — OX, OXX, OXXO, XOX, OXXX (five — no OXXXX/OXXXXX window variant, and no window OXO, confirmed against Duce's actual price list, §7). Same builder, `productClass:'window'`.
- **Casement** — `C` (single), `CCset` (two, same hinge direction), `CCpair` (two, mirrored outward), `CCCCset` (**new in batch 16** — four sashes, all same hinge direction, no mirroring; `CASEMENT_HINGE_EDGES['CCCCset'] = ['left','left','left','left']`), `CCCCpair` (four, as two independent mirrored pairs). Window-only.
- **Awning** — A, AA, AAA, AAAA. All sections `hingeEdge:'top'`.
- **Double-hung** — D, DD, DDD. Each unit is its own two-level subtree (a `kind:'split', axis:'h'` internal transom splitting a top `vertical-slider` (`slideDirection:'down'`) from a bottom `vertical-slider` (`slideDirection:'up'`)). Multiple units fold into a row the same way every other family's sections fold. Unit width, top-pane height, and transom thickness are each one shared value across all units. The between-unit mullion-thickness input only appears when unit count > 1.
  - **Known pending correction, not yet built (flagged again in batch 16, still outstanding):** the internal top/bottom division is currently a plain solid `kind:'split'` transom. Per dimensional data captured in a companion session (a "Casement/Double-hung/Sliding addendum" the person has, not yet folded into this doc's own text), this is wrong — the top sash's own bottom rail and the bottom sash's own top rail (60mm each) genuinely interlock, and this should be `kind:'sashSplit', fromEdge:'start', overlapMM:60`, the same mechanism as §2.2's overlap-sash. **This has not been implemented in this working tree as of batch 16** — confirmed by reading the actual file before starting batch 16 (it still builds a plain split). Batch 16 deliberately did not touch this (a structural change, kept separate from batch 16's routine additions per the project's own single-concern-per-batch practice), and batch 16 also deliberately skipped double-hung's own per-edge sash-shape defaults as a result (see §2.5) — applying 60/60/60/86 to double-hung now would guess at which edge is the "meeting rail" (60mm) vs "outer" (general pattern) before the structural change that actually distinguishes them exists. Needs its own dedicated batch.
  - Similarly pending: sliding assemblies' between-panel overlap has real data now (60mm, X-axis) per the same companion session, but adopting it (`overlapMM` on a `sashSplit` in the sliding builder) is also a structural change to `buildAssemblyPresetTree`'s current plain-`split` mullion mechanism, not built in batch 16, and not attempted here.

**Reopening an assembly to edit its widths:** every family except double-hung stamps `assemblyPresetRef: {pattern, productClass}` on its top-level split at creation, and an "Edit assembly widths" toolbar button reopens the original form pre-filled with current live widths and the marked split's own current mullion thickness. Confirming a width edit preserves every section's existing leaf data exactly — only the split chain's positions/thickness change — validated via `validateTreeAgainstSize`. Double-hung deliberately does not support this yet (its two-level shape doesn't fit the flat-row reopen assumption); the button is genuinely absent, not disabled, for a double-hung selection.

**Catalog-informed default section widths (batch 15):** `ASSEMBLY_PRESET_CATALOG_SIZES`, transcribed from Duce's own NGR price list (§7), keyed by preset **id** (not pattern — `'OX'` the sliding door and `'OX-win'` the sliding window share a pattern string but have different real catalog sizes). On fresh creation only, the form finds the closest real catalog total size to the selected region's actual dimensions and pre-fills each section's width as an equal split of that catalog total. Patterns with no catalog entry (`CCCCpair`, `AA`/`AAA`/`AAAA`, window `OXO`) fall back cleanly to blank/manual entry.

**Mullion-thickness prefill = jamb width (batch 16):** on fresh creation only (never on reopen, which already pre-fills from live tree state), the between-section/between-unit mullion input now defaults to the frame's current `jambL` value (falling back to the pre-existing flat `45` when no outer frame has been added, since there's no jamb to derive from) instead of always defaulting to `45`. This is a **prefill only** — the field remains independently editable, exactly like the batch-15 catalog-width prefill. Applies to every horizontal-row family (sliding/casement/awning) and to double-hung's own between-unit mullion input (a real mullion, in scope even though double-hung's internal structural change is tracked separately) — **not** to double-hung's transom-thickness input, which stays at its own flat default; there is currently no vertically-stacked preset in the tool for a "transom = head width" counterpart rule to apply against, so that's left as a code comment for later, not built.

## 3. Real bugs found and fixed during this build (selected)

1–8. Carried from earlier batches (frame-border overflow, fixed pixel scale, origin-click on frame members, double-click-edit propagation, fixed-panel band survival across a rename, the ancestor-hinge-symbol walk() gap, two separate louvre blade-line bugs, awning-symbol over-suppression, slide-arrow CSS positioning, assemblyPresetRef ancestor-search direction, the catalog id-vs-pattern collision). See prior batch write-ups for full detail on each.

9. **Batch 16 — no new application bugs found; one process gap identified and corrected before writing code.** The batch prompt as first given assumed a `Configurator_Project_Summary_v2.md`, a companion addendum document, and a jsdom-based DOM test harness/suite — none of which existed in this working tree or environment. Rather than inventing plausible-looking substitutes, this was flagged and the actual document contents were requested and supplied inline. Separately confirmed via `node --version`/`pip install selenium` that this environment has no Node.js, no browser automation tooling, and no outbound network access for `pip` — meaning the jsdom/`.click()`-driven test methodology described in the addendum's own §9 could not be reproduced here. Verification for this batch used the same method as batches 1–15 before the addendum's testing methodology was adopted elsewhere: an esprima-based syntax check plus manual/traced-value verification of the actual logic (concrete section counts, hinge-edge arrays, sash-edge shapes, and jamb-prefill scenarios hand-traced against the real code paths) — not a claimed DOM-test assertion count, since none were actually run.

## 4. Component visual language

Reference schematics originally supplied by the user; some were later found to be deliberately simplified relative to real CAD drawings. Colour values: frame `#f47920`, glass `#5be3ea`.

Hinge symbol: two full-length SVG lines, apex away from the hinge edge. Louvre: horizontal blade lines confined to the actual glazed inset. Sliding/vertical arrows: single glyph per direction, positioned within the pane's own extent.

## 5. Known gaps and open items

- No enforcement that a plain `fixed` panel always sits next to a real member.
- No hardware/manufacturability constraints beyond `MIN_PANE_MM = 50`.
- No connection to the AS 1288 pathway engine, Flask app, or any backend — architecture decided (§6), one piece (§6.3's derivation rule) built and tested as a standalone artifact **outside this repo**.
- No drag-and-drop.
- No export schema yet.
- Timber-infill door panels — fully specified in a prior session, not built; parked.
- **Double-hung's internal top/bottom division should be a `sashSplit` overlap (60mm), not a plain `split` transom — real dimensional data exists for this, but it has not been implemented in this repo as of batch 16.** Needs its own dedicated single-concern batch (see §2.9).
- **Sliding assembly overlap (60mm, X-axis) has real data but has not been adopted into `buildAssemblyPresetTree`** — also a structural change, also not attempted in batch 16.
- Double-hung reopen-to-edit — still excluded from the reopen mechanism.
- Sash width per product — to be supplied from CAD drawings, not yet available.
- Depth/z-axis data (frame depth, sash depth, glass/IGU thickness) — specified (§8), not yet built into the data model.
- IGU/glass thickness standard set — confirmed to be a small fixed set, actual values still TBD.
- **No test framework or DOM-automation tooling exists in this working environment** — verification here has consistently relied on static syntax checking (esprima) plus manual trace-verification against concrete inputs, not real browser/DOM-driven tests. If a jsdom-based suite exists elsewhere (per the addendum's §9), it is not present in, or runnable from, this repo/environment as of batch 16.

## 6. AS 1288 human-impact integration — architecture decided, partially built

### 6.1 — Core architectural decision: derivation lives in the export layer only

Every derivation rule below is computed only when preparing data for the AS 1288 engine. The configurator's own live `productClass` and on-screen D/W display are never touched. None of this is built into `configurator_prototype_oxxo_8.html` itself.

### 6.2 — What the configurator already provides, usable as-is

Every pane the configurator produces is already structurally "fully framed" by construction. Glazing method mapping: every configurator type except `louvre` maps to one engine bucket (`'fixed-sash'`); `louvre` maps to its own (`'louvre'`), carrying `bladeWidthMM`/`bladeLengthMM`. Per-pane isolation: no opening-level grouping needed for export. Glazed dimensions: `glazedAreaM2`/the four per-edge sash values directly supply real glass width/height.

### 6.3 — Door / side-panel / window classification — a real derivation rule, fully specified

`productClass:'side-panel'` is never user-set — derived entirely at export time, from adjacency to a door. (Full rule: a sliding door's immediately-adjacent fixed panel in the direction of travel is absorbed as part of the door; any panel beyond that goes through a real geometric side-panel test — within 300mm of the door's own edge, at/below 1200mm height; a hinged or fixed-type door skips absorption entirely; a door panel is never itself reclassified.)

### 6.4 — Standalone rule tester

`side_panel_rule_tester.html` — a self-contained test harness validating the §6.3 rule, built and verified against nine scenarios in a separate session. Not part of this repo.

### 6.5 — FFL (floor level)

Entered once, in the AS 1288 human-impact form itself — not built into the configurator.

### 6.6 — Explicitly still out of scope for v1

Sashless glazing, partly-framed/unframed/butt-joint glazing, any actual wiring into the real Flask app/engine code.

## 7. Reference data captured from Duce's own NGR price list

`NGR_Print_Copy.pdf` — a New Guinea Rosewood timber pricing/costing catalog (Duce Confidential Price List, current as at 01/10/2025).

Frame depth: 140mm or 168mm standard for ordinary single/double-panel products; 210mm/270mm/322mm for progressively wider sliding door assemblies. Sash depth options: 35mm, 42mm, 55mm. Hardwood sill sizes (width × depth, mm): 140×40, 140×68, 168×40, 168×68, 190×40, 195–240×40, 245–290×40, 190×68, 195–240×68, 245–290×68.

Confirmed real Duce product codes: OX/OXX/OXXO/XOX/OXXX/OXXXX/OXXXXX for sliding doors, a narrower five-variant set for sliding windows.

Standard total assembly sizes exist for single/pair casements, single awnings, single/double-unit double-hungs, and every sliding door/window pattern — embedded in `configurator_prototype_oxxo_8.html` as `ASSEMBLY_PRESET_CATALOG_SIZES`.

Sash width is not in this document — to be supplied from CAD drawings.

### 7.1 — Casement / double-hung / sliding window dimensional addendum (captured 16 September 2026, in a session not present in this repo's own history)

Captured from user-supplied AutoCAD elevation screenshots. Door panel data drafted separately, deliberately excluded here.

| Family | Top rail | Bottom rail | Left/right stile | Mullion |
|---|---|---|---|---|
| Casement | 60mm | 86mm | 60mm each | None shown in any reference drawing; a real job's mullion = frame's jamb width |
| Double-hung | 60mm | 86mm | 60mm each | Between units = jamb width |
| Sliding window | 60mm | 86mm | 60mm each | Not required — jambs/interlocks handle the join |

**`PRESET_SASH_EDGES` (batch 16) is sourced directly from this table's casement/awning rows** ("Section... same as awning" per the source), applied to casement and awning assembly-preset leaves only.

**Double-hung internal division — data-model correction, not yet implemented (see §2.9, §5):** the top sash's own bottom rail (60mm) and the bottom sash's own top rail (60mm) genuinely interlock — this should be `kind:'sashSplit', fromEdge:'start', overlapMM:60`, not the current plain `kind:'split'` transom.

**Sliding window interlock/overlap: 60mm, X-axis** — baked into each panel's stated 600mm nominal width, not a separate dimension. This is the real value for the `overlapMM` parameter that has been running on a placeholder (`DEFAULT_OVERLAP_MM = 25`, only ever exercised by the older manual "Add sash (overlap)" mechanism, §2.2) — adopting it into the sliding assembly-preset family (`buildAssemblyPresetTree`) is a structural change (introducing `sashSplit` where that builder currently only ever produces plain `split` mullions), not built in batch 16.

Sliding window's own sash Z-axis depth — still open, explicitly not assumed equal to awning's 35mm.

**Preset naming, confirmed:** "Casement Pair" (mirrored) → `CCpair`; "Casement Set of 2" (same direction) → `CCset`; "Casement Set of 4" → **new**, `CCCCset` (built in batch 16); "Casement Pair of 4" → `CCCCpair`.

## 8. Depth / z-axis data model — decided, not yet built

Resolved to a data problem, not a rendering problem, after clarifying the actual motivation (capturing pricing's missing z-axis dimension). A full 3D model was explicitly rejected.

What's specified but not yet built: `sashDepthMM` (one value per pane, `{35, 42, 55}`); frame depth (one shared value for the whole outer frame, using the 140/168/210/270/322mm figures from §7); glass/IGU thickness (one value per pane, a fixed standard set, values still TBD). None of the three are derived from each other in this version — captured as independent manual inputs, revisit once real source data for that relationship exists.

## 9. Testing methodology

**Batches 1–15 (this repo):** verified via a Python `esprima`-based syntax check of the embedded `<script>` block after every change, plus manual/hand-traced verification of the actual logic against concrete inputs (tree shapes, geometry, validation thresholds) — no DOM automation, no headless browser, no Node.js available in this environment.

**A separate, more thorough methodology (jsdom + real `.click()`/`change`-event DOM interaction, described in a companion addendum) was developed in a different session** and is not present in, or runnable from, this repo. Attempting to install browser-automation tooling here (`pip install selenium`) succeeds with no error but installs nothing (no outbound network access), and no Node.js/npm binary exists on this machine. Until that gap is closed (either by making that tooling available here, or by this repo adopting some other real-interaction test method that *does* work in this environment), batches built here will continue to be verified by the syntax-check + manual-trace method, and this should be stated plainly in each batch's own summary rather than implied to match the addendum's own (unreproduced) methodology.

## 10. Working style this was built with

Chat for architecture/prompt-drafting, Claude Code for real implementation, each batch verified before being trusted. Batches are kept small and single-concern where possible — bundling a novel structural change alongside routine work has reliably produced more correction rounds than splitting them would have (confirmed again in the batch-16 prompt itself, which deliberately separated the double-hung/sliding overlap structural changes from this batch's routine additions).

*End of document.*
