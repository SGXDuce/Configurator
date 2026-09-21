# Door Panel Catalog — Reference Data (extracted from doors.docx)

Extracted directly from the embedded drawings, not the surrounding text —
numbers below are read off each drawing. Where a drawing's own dimensions
don't sum exactly to the stated overall size (rounding in the source
drawing), the discrepancy is noted rather than silently corrected.

**Base sizes found:** most doors (D1–D20, D24, D25, D32) are drawn at
600×2100mm. D22–D24 vary. D21, D23, D26–D31 (the "D21 to D31" range) and
the 63–67 group are drawn at 620×2040mm. Treat the drawn size as the
starting/reference size only — actual size varies per the price list,
not yet supplied.

**Structural families actually present (more than the original 3-pattern
read suggested):**

- **A — Single lite.** One glazing/panel area between stiles and rails.
- **B — Stacked fixed lites.** Multiple lites in a single column,
  separated by full-width rails, all fixed (not swappable).
- **C — Grid (rows × columns).** Lites divided by both horizontal AND
  vertical glazing bars, forming a real grid — not just a single column.
- **D — Interchangeable-zone doors.** One or more zones (shown blue in
  the source) that can be glazing or a named panel material, mixed with
  zero or more always-fixed lites in the same door.
- **E — Solid decorative panel.** A zone that is timber with a routed/
  grooved pattern, no glass, no interchange option at all (D32's lower
  half).
- **F — Vertical-glazing-bar single lite.** One tall lite subdivided
  into multiple columns by thin (23mm) vertical bars only — no rails
  between the columns, no rows.

## D1–D13 — mostly families A, B, C, F (fixed glass throughout)

| id | Size (W×H) | Stile | Top rail | Structure |
|---|---|---|---|---|
| D1 | 600×2100 | 110 | 110 | A — one lite, ~1610mm tall (bottom rail ~190) |
| D2 | 600×2100 | 110 | see structure | B — top lite 1090, mid rail 140, bottom lite 570, bottom rail 190. Content width confirmed 380mm (matches every sibling door) — the drawing's own 398.9mm figure was a stray CAD-export dimension, not a real different stile width. |
| D3 | 600×2100 | 110 | 110 | B — 3 lites of 584.7, 2× 23mm horizontal glazing bars |
| D4 | 600×2107.2 | 110 | 110 | B — 4 lites of 432.8, 3× 23mm bars |
| D5L | 600×2100 | 110 | 110 | B — 5 lites of 341.6, 4× 23mm bars |
| D5 | 600×2100 | 110 | 110 | **Mixed** — top lite 277mm split into 2 columns (178.5 each) by a vertical 23mm bar, one full-width centre lite 1200mm, bottom lite 277mm also 2-column split. Not a uniform grid. |
| D6 | 600×2100 | 110 | 110 | C — 3×2 grid, rows 584.7, columns 178.5, bars 23mm both axes |
| D7 | 600×2100 | 110 | see structure | **Mixed** — fully resolved. Vertical (top to bottom): 110 rail + 100 secondary rail + 23 bar + 1554 main lite + 23 bar + 100 secondary rail + 190 bottom rail = 2100 exact. Horizontal split of the main lite: 100 \| 23 bar \| 257 (asymmetric 2-column) = 380 exact. |
| D8 | 600×2100 | 110 | 110 | C — 4×2 grid, rows 432.8, columns 178.5 |
| D9 | 600×2100 | 110 | 110 | C — 5×2 grid, rows 341.6, columns 178.5 |
| D10 | 600×2100 | 110 | 110 | C — 6×2 grid, rows 280.8, columns 178.5 |
| D11 | 600×2100 | 110 | 110 | C — 4×3 grid, rows 432.8, columns 111.3, bars 23mm both axes |
| D12 | 600×2100 | 110 | 110 | C — 5×3 grid, rows 341.6, columns 111.3 |
| D13 | 600×2100 | 110 | see structure | **Mixed** — fully resolved, identical vertical stack to D7 (110+100+23+1554+23+100+190 = 2100 exact). Horizontal split of the main lite: 100 \| 23 bar \| 134 \| 23 bar \| 100 (symmetric 3-column) = 380 exact. |

## D14–D20 — family D (interchangeable zones), all 600×2100, stile 110

Material options for this range: glazing, MTP, SML (Moulds & panels),
MLDS (Moulds & panels) — per the doc text.

**Bottom interchangeable pair, common to all 7 doors in this range —
resolved:** content width 380mm = 135mm (left interchangeable zone) +
110mm (fixed mullion) + 135mm (right interchangeable zone). Height 570mm
for the pair, preceded by a 140mm fixed rail.

| id | Fixed lite(s) above the mullion | Mid rail | Interchangeable pair |
|---|---|---|---|
| D14 | 1090 (top, full width, single column, fixed-glass) | 140 | 135 \| 110 mullion \| 135, height 570 |
| D15 | 2× 533.5 (stacked, full width, fixed-glass), 23mm bar between | 140 | same |
| D16 | 3× 348 (stacked, full width, fixed-glass), 2× 23mm bars | 140 | same |
| D17 | 2× 533.5, EACH split into 2 columns of 178.5 by a 23mm bar | 140 | same |
| D18 | 3× 348, EACH split into 2 columns of 178.5 | 140 | same |
| D19 | 4× 255.25, EACH split into 2 columns of 178.5 | 140 | same |
| D20 | 255.25 (2-col split) / 533.5 (full width) / 255.25 (2-col split), stacked, 23mm bars between all three | 140 | same |

Note: in every D14–D20 drawing, only the bottom pair is ever
interchangeable — every lite above the mid rail is always fixed glass,
never swappable. Confirmed deliberate.

## D21–D31 — family D, 620×2040 base (only D21–D25, D32 actually drawn; D26–D31 missing, per your own note in the doc text)

Material options for this range: Timber, Plyplanel (10mm ply G2S),
Plypanel (10mm+4mm ply) — per the doc text. You've confirmed glazing is
also a valid option here, same as D14–D20.

| id | Size | Stile | Structure |
|---|---|---|---|
| D21 | — | — | **No drawing in this document** — referenced in the text, not provided |
| D22 | 620×2040 | 110 | D — 2 stacked zones, BOTH interchangeable (1090 top, 570 bottom), mid rail 140 |
| D23 | 620×2040 | 110 | D — 4 stacked zones, ALL interchangeable, 352.5mm each, 110mm rails between every zone |
| D24 | 600×2100 (different base — flag this) | 110 | D — 2 stacked zones, both interchangeable, but each is itself a 2-column side-by-side pair (135mm pitch) — 4 interchangeable sub-zones total |
| D25 | 600×2100 (different base again) | 110 | D — 2 stacked zones, both interchangeable, each a 2-column pair, 490 top / 1170 bottom (uneven split) |
| D26–D31 | — | — | **No drawings in this document** — referenced in the text, not provided |
| D32 | 600×2100 | 110 | **Family E** — top zone fixed glass (1090, not interchangeable), bottom zone (570) is a solid decorative timber panel with routed vertical grooves — no glass, no interchange option shown at all |

## D63–D67 group — mixed, some family F, some family B

Naming convention confirmed from the drawings: **V = vertical glazing
bars, H = horizontal lite rows.** These are NOT all the same structural
family — worth not treating this group as one thing.

| id | Size | Stile | Structure |
|---|---|---|---|
| D63V | 620×2040 | 110 | F — resolved: stile 110 \| glass 60 \| bar 110 \| glass 60 \| bar 110 \| glass 60 \| stile 110 = 620mm exact (3 glass strips, 2 vertical bars, NOT 3 bars — an earlier 3-bar reading was 110mm over) |
| D64H | 620×2040 | 110 | B — 4 fixed horizontal lites of 352.5mm, full 110mm rails between (not thin bars) |
| 65V | 820×2040 | 110 | F — wide door, vertical bars, 4 columns |
| 65H | 620×2040 | 110 | B — 5 fixed horizontal lites of 260mm, 110mm rails |
| 66V | 920×2040 | 110 | F — wider variant, 6 columns of 52mm. Confirmed: this is genuinely "66V", not a duplicate "65V" label — earlier flagged as an inconsistency, now resolved. |
| 66H | 620×2040 | 110 | B — 6 fixed horizontal lites of 198.3mm |
| 67H | 620×2040 | 110 | B — 8 fixed horizontal lites of 154.3mm |

## Flags — things to resolve before this data is trusted as final

1. ~~D7 and D13's column math~~ — **resolved**, see their entries above.
   Both confirmed to sum exactly against the stated overall dimensions.
2. ~~The duplicate "65V" label~~ — **resolved.** The second entry is
   genuinely "66V", not a duplicate of 65V at a different width.
3. **D21 and D26–D31 have no drawings in this document at all** — you
   already knew this; listed here so it's in the same place as the rest
   of the data rather than only in chat history.
4. **D5, D7, D13 don't fit the clean "uniform grid" shape** that most of
   the rest of the range does — they're asymmetric one-offs. Worth
   knowing the catalog needs to support genuinely irregular layouts, not
   just N-rows-by-M-columns.
