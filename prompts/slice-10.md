# Slice 10: Signed coordinate system + directional map expansion

## How to work this file

This slice has three stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation ("go", "yes", "proceed") before starting the next stage.
   If I ask for fixes instead, address those and then ask again.

Do NOT proceed past a stage without explicit confirmation. Do NOT batch stages.
Do NOT rewrite the whole `index.html`. Make surgical edits only.

If a stage's implementation looks like it will exceed ~600 lines of additions,
pause partway through and check in before continuing.

## Committing

Do NOT run git commands. I will handle all commits myself after reviewing each stage.

## Slice context

Refactor the existing Map tab (built in slice 9) to use a **signed, bounded coordinate system** centered on `(0, 0)`.

**Intent:**
- Hex `(0, 0)` is the conceptual center of the world — typically the starting town or campaign home base. It's the default selected hex on first load of a fresh map.
- Coordinates can be negative. Columns run from `colMin` (negative) to `colMax` (positive); same for rows.
- The map can be expanded in any direction independently — "add 10 hexes to the east," "add 5 hexes north," etc. — rather than just setting total cols/rows.

**Existing data:** per user decision, the existing map grid will be wiped on migration. The migration resets `state.hexes` to empty and seeds with defaults. User will repaint.

**Out of scope for this slice:**
- Overlays (rivers/roads) — still deferred to a future slice.
- Local maps / pins — still deferred.
- Pan/zoom — still deferred.
- Multiple maps / campaigns.
- Changing origin after creation (0,0 is always 0,0 once set).

## Schema changes (v5 → v6)

Replace the current `mapMeta` shape with:

```js
mapMeta: {
  colMin: -11,   // inclusive, signed integer
  colMax: 10,    // inclusive
  rowMin: -8,
  rowMax: 7,
  hexSize: 32,
}
```

Defaults on fresh install: `colMin: -11, colMax: 10, rowMin: -8, rowMax: 7` (22×16 total, centered on 0,0).

Migration v5 → v6:
- Replace existing `mapMeta.cols/rows` with new signed bounds as above.
- Reset `state.hexes = {}`.
- Bump `schemaVersion` to 6.
- Also clear `currentHexKey` on every player (set to null) since the old coord values are now meaningless.

Update `docs/data-model.md`: MapMeta entity definition, migration notes, and a sentence in the Hex section clarifying that keys can contain negative integers.

---

## Stage 1 — Schema migration + geometry update

**Schema and migration:**
- Implement the v5 → v6 migration exactly as described above.
- Bump `schemaVersion` to 6.
- Update `docs/data-model.md` with the new `MapMeta` shape, migration steps, and the hex-key-can-be-negative note.

**Geometry helpers:**
Update the existing helpers (from slice 9) to handle signed coordinates:

- `hexCenter(col, row)`: currently assumes `col` and `row` are non-negative indices. Update so it calculates the pixel position relative to `mapMeta.colMin` and `mapMeta.rowMin` — e.g. a hex at `(colMin, rowMin)` renders near canvas origin, a hex at `(0, 0)` renders offset by `(-colMin) * HW` horizontally and `(-rowMin) * VS` vertically.
- `pxToHex(px, py)`: must return keys in the range `[colMin, colMax] × [rowMin, rowMax]`, including negative values. Clicks outside that range return `null`.
- Canvas size: derived from `(colMax - colMin + 1)` columns and `(rowMax - rowMin + 1)` rows.
- The drawing loop (`drawWorld`) iterates `for col = colMin to colMax` and `for row = rowMin to rowMax`.

**Default selection on fresh map:**
On first load of a fresh map (no hex has ever been selected), set `selHex = {col: 0, row: 0}` and render the info panel for that hex.

**Out of scope for this stage:** do not change any UI (toolbar, info panel, settings modal). Only the schema, migration, geometry, and default-select behavior. Everything else still works as in slice 9.

Summarize what you changed. Also explicitly note: "the existing map settings modal still uses the old `cols/rows` shape — this will be addressed in Stage 2."

---

## Stage 2 — Directional expansion UI

Replace the existing "Map Settings" modal (from slice 9 Stage 4) with a new **Expand Map** modal.

**UI:**
The modal shows current bounds at the top as read-only labels:
- "West edge: colMin" (e.g. "West edge: -11")
- "East edge: colMax" (e.g. "East edge: 10")
- "North edge: rowMin"
- "South edge: rowMax"
- "Total size: (colMax - colMin + 1) × (rowMax - rowMin + 1)"

Below, four directional expansion inputs, each a small number input with +/- buttons:
- "Expand west by: [N] hexes" (decreases colMin by N)
- "Expand east by: [N] hexes" (increases colMax by N)
- "Expand north by: [N] hexes" (decreases rowMin by N)
- "Expand south by: [N] hexes" (increases rowMax by N)

Each direction allows negative values (shrinking). Shrinking from any direction that would delete painted hexes shows a warning: "This will delete N painted hexes on the [direction] side. Continue?" Only delete hexes whose `terrain !== 'unknown'` — unknown hexes are "not painted" and can be cleared silently.

Bounds constraints:
- After any expansion, `colMax - colMin + 1` must be between 5 and 120. Same for rows.
- `hexSize` is shown as a separate input (min 16, max 64) and can be changed freely — this doesn't delete any hexes.
- If constraints would be violated, disable the "Apply" button and show an inline error.

**On "Apply":**
- Compute the new bounds.
- Delete painted hexes that fall outside the new bounds (if any).
- Update `mapMeta`.
- Save, redraw canvas. The previously-selected hex, if still in bounds, stays selected; otherwise deselect.
- Close modal.

**On "Cancel":**
- No changes.

**Also update:**
- The toolbar button previously labeled "New Map" should be relabeled "Resize Map" (or similar — the modal is no longer for creating from scratch, just expanding/contracting).
- Remove any old "cols/rows" controls from the UI.

Summarize.

---

## Stage 3 — Info panel coord display, regression checks

**Info panel tweaks:**
- When hex `(0, 0)` is selected, display its coord label with a subtle highlight/annotation like "HEX 0, 0 — Origin" or "HEX 0, 0 ⬤" (a small marker indicating this is the world-center hex).
- Negative coords should display naturally: "HEX -5, 3" not "HEX (−5), 3" or anything fancy.

**Canvas rendering:**
- Consider adding a subtle visual marker on hex `(0, 0)` itself — a small crosshair, dot, or thin ring inside it — so the user can find the origin when scrolling a large map. Should be visible on any terrain including unknown, but unobtrusive. Optional per your judgment; if it conflicts with existing rendering choices, skip it and note that in the summary.

**Status bar:**
- Hovered and selected hex displays should handle negative coords correctly ("HEX: -3, -2").

**Regression checks — these must all still work after this slice:**
1. Factions tab (Detail and Relations sub-tabs) renders and all features work.
2. Players tab: Tier 1 character sheet displays correctly. **The `currentHexKey` validation field should flag any hex key that doesn't exist in `state.hexes`** — after the migration wipe, every player's `currentHexKey` is null, so initially every field should be empty/clean. Typing "0,0" should validate as OK if the hex exists (it will, since `drawWorld` creates hex records on render via `getHex`). Typing "99,99" should flag red.
3. Sessions tab: CRUD, attendance, content layers, tagged lists all functional.
4. Export: the exported JSON now includes the new `mapMeta` shape and works on re-import.
5. Import: loading a v6-format export restores the map correctly. Loading an older v5 export should fail gracefully (alert the user, don't crash).

**Summarize everything I should test across all three stages.**