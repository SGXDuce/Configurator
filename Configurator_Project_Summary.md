# Window/Door Elevation Configurator — Project Summary
### Version: 2.2 (supersedes v2.1 — covers batches 1 through 18, plus AS 1288 integration planning and the depth/glass data model)
### Last updated: 17 September 2026

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

- **Sliding doors** — OX, OXX, OXXO, XOX, OXXX, OXXXX, OXXXXX. `O` → `fixed`, `X` → `horizontal-slider`. Every section defaults `productClass:'door'`. **Batch 18:** every one of these ids now builds via the same overlap/tuck-in mechanism as sliding windows (see the dedicated write-up below) — this is a deliberate assumption (600/660/60mm overlap/40mm tuck-in), explicitly not confirmed against any real door drawing.
- **Sliding windows** — OX, OXX, OXXO, XOX, OXXX (five — no OXXXX/OXXXXX window variant, and no window OXO, confirmed against Duce's actual price list, §7). `productClass:'window'`. **Batch 17:** the 4 core `-win` ids (`OX-win`, `OXX-win`, `XOX-win`, `OXXO-win`) build with a genuinely different internal structure from a plain flat-row split. **Batch 18:** `OXXX-win` — left on the old plain-split path after batch 17 as an open question — is now folded into the same treatment too, as a consistency call (not independently re-confirmed). Every sliding window and door id now shares one mechanism; see the dedicated "Sliding overlap + frame tuck-in" write-up below.
- **Casement** — `C` (single), `CCset` (two, same hinge direction), `CCpair` (two, mirrored outward), `CCCCset` (**new in batch 16** — four sashes, all same hinge direction, no mirroring; `CASEMENT_HINGE_EDGES['CCCCset'] = ['left','left','left','left']`), `CCCCpair` (four, as two independent mirrored pairs). Window-only.
- **Awning** — A, AA, AAA, AAAA. All sections `hingeEdge:'top'`.
- **Double-hung** — D, DD, DDD. Each unit is its own two-level subtree (a `kind:'split', axis:'h'` internal transom splitting a top `vertical-slider` (`slideDirection:'down'`) from a bottom `vertical-slider` (`slideDirection:'up'`)). Multiple units fold into a row the same way every other family's sections fold. Unit width, top-pane height, and transom thickness are each one shared value across all units. The between-unit mullion-thickness input only appears when unit count > 1.
  - **Known pending correction, not yet built (flagged again in batch 16, still outstanding):** the internal top/bottom division is currently a plain solid `kind:'split'` transom. Per dimensional data captured in a companion session (a "Casement/Double-hung/Sliding addendum" the person has, not yet folded into this doc's own text), this is wrong — the top sash's own bottom rail and the bottom sash's own top rail (60mm each) genuinely interlock, and this should be `kind:'sashSplit', fromEdge:'start', overlapMM:60`, the same mechanism as §2.2's overlap-sash. **This has not been implemented in this working tree as of batch 16** — confirmed by reading the actual file before starting batch 16 (it still builds a plain split). Batch 16 deliberately did not touch this (a structural change, kept separate from batch 16's routine additions per the project's own single-concern-per-batch practice), and batch 16 also deliberately skipped double-hung's own per-edge sash-shape defaults as a result (see §2.5) — applying 60/60/60/86 to double-hung now would guess at which edge is the "meeting rail" (60mm) vs "outer" (general pattern) before the structural change that actually distinguishes them exists. Needs its own dedicated batch.
  - **Sliding overlap + frame tuck-in — implemented in batch 17 (window ids only), extended to every sliding door/window id in batch 18.** `buildAssemblyPresetTree` takes an explicit `id` parameter (not derivable from `pattern`, since `'OX'` the sliding door and `'OX-win'` the sliding window share a pattern string) and branches to `buildSlidingWinOverlapTree` whenever `isSlidingOverlapId(id, pattern)` is true — a gate that (as of batch 18) matches **any** pure-O/X pattern id, door or window, not just `-win`-suffixed ones (renamed from batch 17's narrower `isSlidingWinId`). Every fixed panel (`O`) is 600mm, every sliding panel (`X`) is 660mm (60mm wider — the sliding leaf always reaches over its neighbour, frontmost, never the fixed one). Each adjacent join is classified via `classifySlidingWinJoin` (reusing `computeDefaultSlideDirections`, the existing source of truth for which way each `X` points): an `O`-`X` join is always a 60mm overlap with `X` as the sash; an `X`-`X` join with both `X`s pointing the same direction is a 60mm overlap with the **farther-from-the-reference-O** `X` as the sash (frontmost); an `X`-`X` join with diverging directions is **flush** — no overlap at all, built as an ordinary zero-thickness `kind:'split'` (the pre-existing zero-thickness plain-split mechanism, not a new construct) rather than a sash of zero size, since the two panels genuinely meet stile-to-stile with no interlock. A converging `X`-`X` combination has no confirmed physical meaning and has never occurred in any pattern checked so far (the four batch-17 patterns, plus the longer door-only `OXXXX`/`OXXXXX` hand-traced in batch 18) — `classifySlidingWinJoin` throws rather than guessing if one is ever produced. `buildSlidingWinOverlapTree` and `classifySlidingWinJoin` are purely pattern-driven (only ever read O/X letters, never the id string) — extending them from window-only to also cover doors in batch 18 needed no change to either function, only to the `isSlidingOverlapId` gate deciding which ids reach them.
  
    **Frame tuck-in (batch 18 addition):** any panel edge that touches the outer frame (a jamb, the head, or the sill) has 20mm of its stile/rail hidden inside that frame member's own cavity. For a single horizontal row this is always exactly the two frame-touching edges (leftmost panel's left edge, rightmost panel's right edge) regardless of panel count, so it's a **flat 40mm per row**, not per-panel or per-join — `SLIDING_TUCKIN_MM = 40`. Gated on `hasFrame` (there's no jamb cavity to tuck into with no outer frame added) — **this gating is a default applied per the batch-18 instruction, not independently confirmed by the person; still flagged as open, see §5.** Confirmed as a genuinely separate, additive effect from the 60mm overlap (a hypothetical worked OX example: 2×1220mm sash panels in a 2340mm content width — the 100mm total gap splits into 60mm overlap + 40mm tuck-in, confirmed via exact arithmetic).

    Catalog-informed prefill (§2.9/batch 15) is tuck-in-aware: `computeSlidingWinPrefillWidths` solves `O = (T − 60·n_X + 60·joins [+ 40 if hasFrame]) / (n_O + n_X)`, `X = O + 60` against the real catalog total (`joins` counts only real overlap joins, excluding flush ones). With `hasFrame:false` this reproduces batch 17's original numbers exactly (verified: `OX-win` T=1200 → O=600/X=660, `OXX-win` T=2700 → O=900/X=960, `OXXO-win` T=2400 → O=600/X=660 — all unchanged). **With `hasFrame:true`, the same real catalog totals no longer all land clean** — `OX-win`/`OXXO-win` (2- and 4-panel, so the flat 40mm divides evenly) still land exactly (e.g. `OX-win` T=1200 → O=620/X=680), but every `OXX-win` total needs the existing 5mm-rounding fallback (e.g. T=2700 → raw O=913.33, rounds to 915) — a genuine, worth-noting behavioural difference from batch 17, where the fallback was never exercised by real data at all. `XOX-win` has no catalog entry at all, so it prefills with the plain 600/660 defaults directly (unaffected by tuck-in, since there's no catalog total to solve against). A sliding id has no mullion input at all in its widths form (its joins are overlap/flush, never a separate mullion member) — its entered-total check instead uses `computeSlidingWinTotalWidth` (sum of section widths, minus 60mm per real overlap join, minus the flat 40mm tuck-in when `hasFrame` is true). "Edit assembly widths" reopen is also supported: `assemblyPresetRef` stores `id` (not just `{pattern, productClass}`, since pattern alone can't distinguish a sliding-overlap id from a plain one), and a dedicated `computeSlidingWinSectionPaths` re-derives each section's real tree path (which is **not** the uniform right-fold `[1,1,...,0]` shape every other family uses, since this tree mixes `sashSplit` and flush-split nodes) for both `getAssemblyPresetCurrentState` and `getAssemblyLeafNodesInOrder` — hand-verified correct for the new longer door patterns (`OXXXX`, `OXXXXX`) in batch 18, not just the original four. Catalog sizes remain keyed per literal id, unchanged (`OX` the door and `OX-win` the window still have their own distinct real catalog totals even though they now share the same overlap/tuck-in mechanism) — door and window catalog data is never merged or shared.

    **Door extension is a stated assumption, not confirmed data:** the person confirmed sliding doors should get identical treatment (600/660/60mm overlap/40mm tuck-in) to sliding windows, explicitly as a deliberate modeling assumption for this tool — everything the underlying numbers are built from came from window drawings, not door ones. If a real door drawing later shows different figures, this needs correcting, not defending. This applies to every sliding door pattern, including the door-only-length ones with no window equivalent at all (`OXXX`, `OXXXX`, `OXXXXX`).

**Reopening an assembly to edit its widths:** every family except double-hung stamps `assemblyPresetRef: {pattern, productClass, id}` (`id` added batch 17) on its top-level node at creation, and an "Edit assembly widths" toolbar button reopens the original form pre-filled with current live widths and the marked split's own current mullion thickness (or, for any sliding-overlap id — batch 18: doors included — no mullion at all, see above). Confirming a width edit preserves every section's existing leaf data exactly — only the split chain's positions/thickness (or, for a sliding-overlap id, the whole overlap tree, rebuilt via `buildSlidingWinOverlapTree`) change — validated via `validateTreeAgainstSize`. **Batch 17 note, still true in batch 18:** a sliding-overlap preset's top-level node can itself be a `sashSplit` rather than a plain `split` (e.g. `OX`/`OX-win`/`OXX`/`OXX-win`/`XOX`/`XOX-win`, whose sections form a single overlap run with no flush join at all — only patterns with a middle diverging X-X join, like `OXXO`/`OXXO-win`, produce a plain `split` at the top) — every place that assumes `node.kind === 'split'` to find/stamp/select this marker (`findAssemblyPresetAncestor`, the "selected bar" edit-button check, the marker-stamping in `confirmAssemblyPreset`) also accepts `sashSplit`. Double-hung deliberately does not support reopen yet (its two-level shape doesn't fit the flat-row reopen assumption); the button is genuinely absent, not disabled, for a double-hung selection.

**Catalog-informed default section widths (batch 15):** `ASSEMBLY_PRESET_CATALOG_SIZES`, transcribed from Duce's own NGR price list (§7), keyed by preset **id** (not pattern — `'OX'` the sliding door and `'OX-win'` the sliding window share a pattern string but have different real catalog sizes). On fresh creation only, the form finds the closest real catalog total size to the selected region's actual dimensions and pre-fills each section's width as an equal split of that catalog total. Patterns with no catalog entry (`CCCCpair`, `AA`/`AAA`/`AAAA`, window `OXO`) fall back cleanly to blank/manual entry.

**Mullion-thickness prefill = jamb width (batch 16):** on fresh creation only (never on reopen, which already pre-fills from live tree state), the between-section/between-unit mullion input now defaults to the frame's current `jambL` value (falling back to the pre-existing flat `45` when no outer frame has been added, since there's no jamb to derive from) instead of always defaulting to `45`. This is a **prefill only** — the field remains independently editable, exactly like the batch-15 catalog-width prefill. Applies to every horizontal-row family (sliding/casement/awning) and to double-hung's own between-unit mullion input (a real mullion, in scope even though double-hung's internal structural change is tracked separately) — **not** to double-hung's transom-thickness input, which stays at its own flat default; there is currently no vertically-stacked preset in the tool for a "transom = head width" counterpart rule to apply against, so that's left as a code comment for later, not built.

## 3. Real bugs found and fixed during this build (selected)

1–8. Carried from earlier batches (frame-border overflow, fixed pixel scale, origin-click on frame members, double-click-edit propagation, fixed-panel band survival across a rename, the ancestor-hinge-symbol walk() gap, two separate louvre blade-line bugs, awning-symbol over-suppression, slide-arrow CSS positioning, assemblyPresetRef ancestor-search direction, the catalog id-vs-pattern collision). See prior batch write-ups for full detail on each.

9. **Batch 16 — no new application bugs found; one process gap identified and corrected before writing code.** The batch prompt as first given assumed a `Configurator_Project_Summary_v2.md`, a companion addendum document, and a jsdom-based DOM test harness/suite — none of which existed in this working tree or environment. Rather than inventing plausible-looking substitutes, this was flagged and the actual document contents were requested and supplied inline. Separately confirmed via `node --version`/`pip install selenium` that this environment has no Node.js, no browser automation tooling, and no outbound network access for `pip` — meaning the jsdom/`.click()`-driven test methodology described in the addendum's own §9 could not be reproduced here. Verification for this batch used the same method as batches 1–15 before the addendum's testing methodology was adopted elsewhere: an esprima-based syntax check plus manual/traced-value verification of the actual logic (concrete section counts, hinge-edge arrays, sash-edge shapes, and jamb-prefill scenarios hand-traced against the real code paths) — not a claimed DOM-test assertion count, since none were actually run.

10. **Batch 17 (sliding-window overlap) — two real design bugs caught and fixed during implementation, before either could reach the committed code as silent breakage.**
    - The original batch prompt asserted the overlap-sash's "sash is always `children[0]`" as a fixed geometric rule. Tracing `walk()`'s actual `sashSplit` branch line-by-line showed this only holds for `fromEdge:'start'` — `fromEdge:'end'` puts the sash at `children[1]`. Flagged to the user rather than guessing; the user confirmed the trace was correct and their own original wording was "imprecise to the point of wrong if taken literally" (letting screen position diverge from the O/X pattern string, which is how these patterns are specified and read throughout the project, would have been a real correctness bug). The actual rule implemented: `fromEdge:'start'` when the sash is the physically-left leaf of a join, `fromEdge:'end'` when it's the physically-right leaf.
    - While deriving `computeSlidingWinSectionPaths` (needed so "Edit assembly widths" reopen can find each section's real live node/rect in a `-win` tree, whose shape is not the uniform right-fold every other family uses), an initial version assumed the accumulated-so-far node's path prefix direction flipped depending on `sashSide` (left vs right). Cross-checking against an actual simulated tree build (Python, mirroring `buildRun`'s real `children: [node, newLeaf]` order literally) showed `children[0]` is **always** the accumulated node and `children[1]` is **always** the newly-added leaf, regardless of `sashSide`/`fromEdge` — `fromEdge` only changes which child the *geometry* treats as the sash, never which child a given leaf structurally becomes. The initial version produced wrong paths for `XOX-win` and `OXXO-win` (verified via `get_node(tree, path)` returning the wrong leaf); corrected and re-verified exact for all 4 patterns before trusting it.
    - Separately (not a bug reaching code, but worth recording): a `-win` preset's top-level tree node can be a `sashSplit` rather than a `split` whenever the whole pattern is a single overlap run with no flush join (`OX-win`, `OXX-win`, `XOX-win` — only `OXXO-win`'s middle flush join produces a top-level plain `split`). The existing `tree.kind === 'split'` checks used to stamp/find/select the `assemblyPresetRef` marker were `sashSplit`-blind, which would have silently left 3 of the 4 `-win` ids with no marker at all (no "Edit assembly widths" button, ever) had it not been checked against the actual tree shapes those 3 ids produce before committing. Fixed by widening every such check to accept `sashSplit` alongside `split`.
    - Verification method: same as batch 16 (no jsdom/browser automation available in this environment) — esprima syntax check (pass) plus exhaustive hand/Python-simulated tracing of `classifySlidingWinJoin`, `buildSlidingWinOverlapTree`, `computeSlidingWinSectionPaths`, and `computeSlidingWinPrefillWidths`/`computeSlidingWinTotalWidth` against all 4 confirmed real patterns (`OX-win`, `OXX-win`, `XOX-win`, `OXXO-win`), confirming exact totals (1200/1800/1800/2400mm), exact catalog-formula outputs (e.g. `OXX-win` T=2700 → O=900, X=960), and exact leaf-path resolution for every section of every pattern. This is traced-logic verification, not a real-DOM/jsdom test run — that tooling remains unavailable in this environment, consistent with batch 16's own finding.

11. **Batch 18 (sliding overlap tuck-in correction + extend to sliding doors) — no new bugs in the existing batch-17 code; one real quantitative finding surfaced by testing, not a defect.**
    - `computeSlidingWinTotalWidth`/`computeSlidingWinPrefillWidths` were missing the flat 40mm frame tuck-in term identified in the addendum — a genuine correction to batch 17's formula (a gap, not a reversal; the 600/660/60mm figures stand unchanged). Fixed by adding `SLIDING_TUCKIN_MM = 40`, applied once per row (not per-panel/per-join), gated on `hasFrame` per the batch prompt's explicit instruction — this gating is a default, not independently confirmed by the person (see §5).
    - `isSlidingWinId` (batch 17, matched only `-win`-suffixed ids) was renamed to `isSlidingOverlapId` and broadened to match any pure-O/X pattern id, door or window — every one of the ~16 call sites that gated behaviour on the old narrower check was updated to the new one; verified none were missed via a full-file grep before considering this done.
    - **Verified, not assumed, that `buildSlidingWinOverlapTree`/`classifySlidingWinJoin` generalize to the longer door-only patterns**, per the batch prompt's explicit instruction not to assume this: hand/Python-traced `OXXX`, `OXXXX`, and `OXXXXX` — all joins resolve to `sashSide:'right'` (a single reference `O` at one end means every `X` points the same direction), so neither the converging-join guard nor the "left-sash-after-first-join" defensive guard is ever triggered by these patterns. Both functions are purely pattern-driven (never read the id string), which is exactly why extending them from window-only to door-inclusive needed zero changes to either function — only to the `isSlidingOverlapId` gate.
    - **Quantitative finding, not a bug:** with `hasFrame:true`, the real `OXX-win` catalog totals (2700/3600/4500mm) no longer solve to a clean round `O` under the tuck-in formula (e.g. T=2700 → raw O=913.33mm) — the existing 5mm-rounding fallback (built in batch 17, never previously exercised by real data) now genuinely activates for every `OXX-win` catalog size. `OX-win`/`OXXO-win` (2- and 4-panel, so the flat 40mm divides evenly) still land exactly clean. With `hasFrame:false`, every pattern reproduces batch 17's original clean numbers exactly, confirming the fallback isn't a new problem, just a real consequence of the flat-40mm-not-dividing-evenly-by-3 fact once tuck-in is added.
    - Verification method: same as batches 16–17 (no jsdom/browser automation in this environment) — esprima syntax check (pass) plus exhaustive Python-simulated tracing: (a) all 4 original window patterns re-verified with the `+40` term folded in, both `hasFrame:true` and `hasFrame:false`, confirming `hasFrame:false` reproduces batch 17's numbers exactly and `hasFrame:true` produces the new (documented) totals; (b) `OXXXX`/`OXXXXX` traced end to end — join classification, tree construction, section-path derivation, and total-width arithmetic including both the overlap and tuck-in terms; (c) `OXXX-win` confirmed to now route through `isSlidingOverlapId`'s broadened check rather than the old plain-split fold. This is traced-logic verification, not a real-DOM/jsdom test run.

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
- ~~Sliding assembly overlap (60mm, X-axis) has real data but has not been adopted into `buildAssemblyPresetTree`~~ — **implemented in batch 17** for the 4 core `-win` ids, **extended to every sliding door and window id (including the 40mm frame tuck-in correction) in batch 18** (see §2.9).
- **Batch 18 tuck-in — four items raised while deriving the correction, explicitly left open, none resolved by batch 18 itself:**
  - Whether the 40mm tuck-in should apply at all when `hasFrame:false` (no jamb/head/sill cavity exists to tuck into). Batch 18 implements "no" as the default (tuck-in only applies when `hasFrame` is true) per the batch prompt's own instruction, but this is a default being applied, not a fact confirmed by the person — needs explicit sign-off before being treated as settled.
  - Whether double-hung's tuck-in also applies **horizontally** (the leftmost/rightmost unit in a DD/DDD row tucking into the left/right jamb) — only the *vertical* (head/sill) tuck-in for double-hung was actually discussed and confirmed; horizontal was never raised.
  - Whether this tuck-in mechanism applies to casement or awning at all — not discussed; only sliding (doors+windows) and double-hung (vertical only) were covered. Don't extend it there without asking.
  - Whether the double-hung transom input disappears entirely once the transom→overlap structural conversion lands (matching how a sliding-overlap id lost its mullion input entirely), or stays as an input defaulting to 60mm — asked, never directly answered.
- **Related items raised in the same session, explicitly tracked separately from batch 18's own scope — not yet diagnosed or built:**
  - Casement's mullion-optional UI (an explicit with/without prompt after preset selection).
  - `findClosestCatalogSize` comparing a content-area rect against outer-frame catalog sizes with no reduction — a real mismatch bug, cause understood, fix not yet built.
  - Awning/double-hung's own missing catalog-informed prefill (batch 15 only covers sliding/casement) — cause not yet diagnosed.
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

**Sliding window interlock/overlap: 60mm, X-axis** — baked into each panel's stated 600mm nominal width, not a separate dimension. This is the real value for the `overlapMM` parameter that has been running on a placeholder (`DEFAULT_OVERLAP_MM = 25`, only ever exercised by the older manual "Add sash (overlap)" mechanism, §2.2) — adopted into the sliding assembly-preset family in batch 17 (window ids), extended to doors in batch 18.

### 7.2 — Sliding overlap + frame tuck-in addendum (captured 17 September 2026, working forward from a hypothetical worked example, not a real Duce drawing)

Re-derives the sliding overlap math after a hand-worked OX example (600×2400mm outer, head/jamb=30mm, sill=40mm, sash rails/stiles=60mm each) didn't balance under the batch-17 formula alone. Two separate, additive effects, confirmed distinct via exact arithmetic (2×1220 − 2340 = 100 = 60 overlap + 40 tuck-in):

1. **The 60mm overlap** (batch 17, unchanged, not reversed) — the sliding panel's stile fully covers the fixed panel's stile where they meet.
2. **Frame tuck-in (new in this addendum, batch 18)** — any panel edge touching the outer frame (jamb, head, or sill) has 20mm of its stile/rail hidden inside that member's own cavity. For one horizontal row this is always exactly two edges (leftmost panel's left edge, rightmost panel's right edge) regardless of panel count — a **flat 40mm per row**, confirmed explicitly not per-panel/per-join. Confirmed to also apply vertically to double-hung (top sash tucks 20mm into the head, bottom sash 20mm into the sill) "for the purpose of this configurator" — the person's own words, i.e. a deliberate modeling assumption, not independently verified against a real double-hung section drawing.

Sliding doors extended to the identical treatment (600/660/60/40mm), explicitly stated as an assumption, not confirmed from any door-specific drawing — covers every door pattern including the door-only lengths (`OXXX`/`OXXXX`/`OXXXXX`). `OXXX-win` folded in too, as a consistency call, flagged rather than independently re-confirmed.

Four items raised in this session were left explicitly unresolved — see §5's own batch-18 bullet for the full list (the `hasFrame` gating default, double-hung's horizontal tuck-in, casement/awning applicability, and the double-hung transom-input question).

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
