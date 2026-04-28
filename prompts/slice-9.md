# Slice 9: World Hex Map — grid, terrain, hex metadata

## How to work this file

This slice has four stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation before starting the next stage.
   Confirmations will be short ("go", "yes", "proceed", "continue").
   If I ask for fixes instead, address those and then ask again.

Do NOT proceed past a stage without explicit confirmation. Do NOT batch stages.
Do NOT rewrite the whole `index.html`. Make surgical edits only.

If a stage's implementation looks like it will exceed ~800 lines of additions,
pause partway through and check in before continuing.

## Context

Port the world-hex-grid portion of `reference/hexmap.html` into the Map tab of the merged app.

**Method:** surgical additions to `index.html`. No rewrites. String-replace-style edits. If existing code needs modification, call it out explicitly.

**Stages:** four. I'll send them one at a time. Implement each, summarize, wait for me to test before the next.

**Scope (whole slice):**
- World hex grid canvas, scrollable container (no pan/zoom).
- Configurable grid dimensions via a map-settings modal, stored in `state.mapMeta`.
- Terrain painting across 11 terrain types (plains, forest, dark-forest, hills, mountain, water, swamp, desert, ruins, settled, unknown).
- Explored flag per hex.
- Hex name and notes, editable via right-side info panel.
- Tools: paint, erase, mark explored, edit note.
- Export / import map data as JSON.

**Out of scope (deferred to later slices):**
- Overlays (rivers, roads, edge-picker modal).
- Local maps per hex.
- Pins of any kind.
- Pan/zoom, mini-map.
- Travel-time calculations.
- Cross-links to factions, NPCs, sessions, rumors.

**Layout:**
- Top toolbar: terrain brush palette (11 swatch buttons), tool mode buttons (Paint / Erase / Explored / Note), action buttons (New / Export / Import). Compact, single row where possible. Flow to a second row if needed on narrow viewports.
- Main canvas area: scrollable container, canvas inside. Canvas sized to grid dimensions.
- Right info panel (~240px wide): shows selected hex's coord, terrain, explored toggle, name input, notes textarea, save button. Empty-state when no hex selected.
- Status bar at the bottom: current mode, brush, hovered/selected hex coord.

**Styling:** match the existing app. Section-box pattern, Cinzel labels, Crimson Pro body text, gold accents. The reference file's terrain color variables (`--t-plains`, `--t-forest`, etc.) should be copied into the main `:root` block since those are what the canvas actually uses.

**Schema change (v4 → v5):** add `state.mapMeta` with `{ cols: 22, rows: 16, hexSize: 32 }` as defaults. Migration sets these values if missing. Bump `schemaVersion` to 5.

Ready for Stage 1?

---

## Stage 1 — Scaffolding and schema migration

- Schema migration v4 → v5: add `state.mapMeta = { cols: 22, rows: 16, hexSize: 32 }` if missing. Set `schemaVersion = 5`. Update `docs/data-model.md` — add a `MapMeta` entity definition and note the migration.
- Verify `state.hexes` object exists (default `{}`).
- Add "Map" tab button to the header tab bar, placed first (before Factions) since the map is the most spatially-primary view.
- Add the Map tab content panel with:
  - Top toolbar (placeholder: just a row with "Map" label for now).
  - Main canvas area (scrollable container with a `<canvas id="map-canvas">`).
  - Right info panel (placeholder: "No hex selected").
  - Status bar (placeholder: "MODE: — | BRUSH: — | HEX: —").
- Canvas is sized from `state.mapMeta` on render. Draw nothing yet — just a blank canvas with the correct pixel dimensions.
- No interactions yet.

Summarize what you changed.

---

## Stage 2 — Hex grid rendering + terrain brush + paint mode

- Constants: `SQRT3 = Math.sqrt(3)`, `HW = SQRT3 * hexSize`, `VS = hexSize * 1.5`, `PAD = 10`. Canvas size derived from `mapMeta` and these.
- Pointy-top hex geometry. Offset rows (odd rows shift right by `HW/2`).
- Helper functions: `hexCenter(col, row)`, `hexCorner(cx, cy, i, size)`, `pxToHex(px, py)` — pick these up from `reference/hexmap.html` but adapt to read `mapMeta` instead of hardcoded constants.
- Terrain definitions: copy `TCOLORS`, `TNAMES`, `TICONS` from the reference file's script. Add the `--t-*` CSS variables to `:root` in the main stylesheet.
- `drawWorldHex(col, row)`: renders one hex with terrain fill, coord label in the top area, terrain icon glyph in the center, hex name (italic) below if set.
- `drawWorld()`: clears canvas, draws background gradient, iterates all `(col, row)` in bounds and calls `drawWorldHex`.
- `getHex(col, row)`: ensures a hex record exists in `state.hexes` with defaults `{ terrain: 'unknown', name: '', note: '', explored: false }`. Note: do NOT add `overlays` or `localPins` yet — those are future slices. Keep the record minimal.
- Top toolbar: terrain brush palette with 11 swatch buttons, each a labeled color swatch matching the reference's sidebar terrain section but laid out horizontally. Selecting a brush updates a `curTerrain` variable and highlights the button. Default selection: "plains."
- Top toolbar: tool buttons. For this stage, just "Paint" (default, active) and "Erase". Clicking a tool sets `curMode` and updates highlight.
- Canvas interaction:
  - Click: in paint mode, set the clicked hex's terrain to `curTerrain`, save, redraw. In erase mode, reset that hex to defaults.
  - Drag (mousedown + move): paint/erase continuously until mouseup.
- Status bar updates live: MODE, BRUSH, and hovered HEX coord.

Summarize.

---

## Stage 3 — Info panel, hex metadata, explored mode

- Selecting a hex (any left-click) sets `selHex = {col, row}` and triggers info-panel render. Selected hex gets a gold border in the canvas.
- Right info panel, when a hex is selected, shows:
  - Coord label: "HEX col, row"
  - Terrain indicator: swatch + name
  - Explored checkbox (toggles `hex.explored`, redraws)
  - "Hex Name" text input (live-bound to `hex.name`, triggers redraw on change so the canvas label updates)
  - "Notes" textarea (live-bound to `hex.note`)
  - A "Save Note" button that acts as a visual confirmation only (data is already saved live; button briefly shows "SAVED ✓" then reverts to "SAVE NOTE"). This is a UX convention from the reference — keep it.
- Explored rendering: when `hex.explored === true`, overlay a subtle gold-tinted radial gradient on the hex fill and draw a thin gold stroke. When `false` and `terrain !== 'unknown'`, darken the fill slightly so unexplored-but-painted hexes look faded.
- A small dot indicator in the corner of any hex with a non-empty note (matches reference).
- Tool buttons gain two more entries in the toolbar: "Explored" (click-to-toggle explored on hexes) and "Note" (click selects and focuses the note field in the info panel, no other effect).
- Status bar shows HEX coord on hover and selection.

Summarize.

---

## Stage 4 — Map settings modal, export/import, final polish

- Add a "Map" section to the toolbar with three buttons: "New Map" (opens settings modal), "Export", "Import".
- **Map Settings Modal**:
  - Inputs for cols (min 5, max 60), rows (min 5, max 40), hex size (min 16, max 64).
  - Pre-filled with current `mapMeta` values.
  - "Create" button.
  - If the user reduces dimensions below current hex data range, show a warning: "Hexes outside the new bounds will be deleted. Continue?" Checkbox or second confirm before actually resizing.
  - On confirm: update `mapMeta`, delete out-of-bounds hex records, clear `selHex` if it's out of bounds, resize canvas, redraw.
  - "Cancel" dismisses with no change.
- **Export**: triggers download of a JSON file containing `{ mapMeta, hexes }`. Filename: `hexmap-export.json` with today's date appended.
- **Import**: file picker accepting `.json`. On selection, parse and validate shape. If valid, ask for confirm ("Replace current map? This cannot be undone."), then load into `state.mapMeta` and `state.hexes`, save, redraw. If invalid, show an error toast or alert.
- **Regression checks**: confirm the Factions, Players, and Sessions tabs still load cleanly. Confirm `currentHexKey` validation on a Player (red border when hex doesn't exist) now actually works — any key matching an existing hex in `state.hexes` should pass, any other key should flag.
- Summarize everything I should test across all four stages.