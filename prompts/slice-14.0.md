# Slice 14: Map overlays (rivers and roads)

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
Do NOT rewrite `index.html` wholesale — make surgical edits only.

If a stage looks like it will exceed ~600 lines of additions, pause partway
through and check in before continuing.

## Slice context

Add map overlays — rivers and roads drawn on the hex map. The schema
already has the scaffolding (`Hex.overlays: [Overlay]` with `type`,
`edges`, `flowFrom`), but no UI exists to create or render them. This
slice builds the rendering, the paint tools, the detail editor, and
the road surface variants.

Schema bumps v9 → v10 — `edges` constraint relaxes from "1 or 2 indices"
to "1–6 indices" (so a single overlay can branch through any combination
of a hex's six edges), and a new optional `surface` field is added for
road material variants.

**Back up `wm_unified_v1` from localStorage before starting Stage 1.**
The migration is trivial (defensive defaults only), but a backup is
cheap insurance.

### In scope

- Schema v10 migration (defensive: ensure `overlays: []` on every hex).
- Relaxed `edges` constraint: 1–6 indices instead of 1–2.
- New `Overlay.surface: "dirt" | "gravel" | "stone"` field (roads only;
  rivers ignore it).
- Rendering: rivers and roads drawn as lines from hex center to each
  listed edge midpoint, forming spokes that meet at the center.
- Map toolbar modes: paint road, paint river, erase road, erase river.
- **Click-to-add**: clicking inside a hex (in paint mode) adds the
  nearest edge to that hex's overlay of that type.
- **Drag-trace**: dragging across hexes (in paint mode) adds the
  entry/exit edges to each hex traversed, building continuous lines
  the same way the terrain painter works.
- **Eraser modes**: click in a hex (in erase mode) removes that overlay
  type from the hex entirely.
- Per-hex detail editor section: checkboxes per edge for road, river;
  flow direction selector; road surface selector.
- Visual: rivers blue with flow arrow at the source edge pointing
  inward; roads in three surface colors (dirt = light brown, gravel =
  gray-brown, stone = pale gray).

### Out of scope — do not build in this slice

- Bridges, fords, tunnels (no schema for these yet).
- Terrain costs / pathfinding (slice 14.5).
- Road overlay influence on faction territory rendering.
- Animated water / current ripples.
- Saving overlay templates / copy-paste.
- Multiple separate road systems crossing the same hex without
  meeting (modeled instead by adjacent hexes — one road overlay per
  type per hex).

### Schema changes

```js
// Hex.overlays element shape (v10)
{
  type:     "river" | "road",
  edges:    [number],           // 1–6 indices in [0..5]
  flowFrom: number | null,      // edge index 0..5 (must be a member of `edges`)
                                //   — for rivers only; null = no flow direction set
                                //   — null for roads
  surface:  "dirt" | "gravel" | "stone",  // roads only; default "dirt"
                                          //   — ignored / not stored on rivers
}
```

**Edge index convention.** For pointy-top hexes, edges 0–5 map to:
`NE / E / SE / SW / W / NW`. Verify this matches whatever slice 9
established — if slice 9 rotated this mapping, use slice 9's rotation
and update the directional labels in the detail editor accordingly.

**Multi-edge semantics.** A single overlay's `edges` array holds all
edges that connect to that hex's overlay of that type. They all meet
at the hex center, forming a star. So:

- Pass-through (2 edges): straight line through the hex.
- Stub (1 edge): half-line from center to that edge.
- Y-junction (3 edges): three spokes meeting at center.
- Up to 6 edges: full hex cluster point.

**Flow semantics for rivers.** `flowFrom` is the edge index (0..5) of
the source edge — the edge through which water enters the hex. It
must be a member of `edges`. The water exits through every other
edge in `edges`. Rendering: arrow at the source edge pointing INTO
the hex; arrows at every other edge pointing OUT of the hex.

If `flowFrom` is null, render the river without arrows.

When edges change (via paint, erase, or detail editor) and `flowFrom`
is no longer a member of `edges`, set `flowFrom = null`.

Tributaries (multiple sources merging) aren't representable in this
model. The 5-mile hex granularity makes it acceptable to model them as
adjacent hexes converging visually rather than at a junction within
one hex.

### State additions

In `state.ui`:

```js
mapMode: 'terrain' | 'paintRoad' | 'paintRiver' | 'eraseRoad' | 'eraseRiver' | null,
//   — extends the existing terrain paint mode field if one exists; if the
//     existing pattern uses a different field name, mirror it instead of
//     introducing this one. Either way, the four new modes must coexist
//     with the existing terrain paint mode.
```

No new persistent state on overlays beyond what's already on the hexes.

### Naming / CSS conventions

- CSS class prefix for overlay-specific styles: `o-*` (e.g. `.o-river`,
  `.o-road-dirt`, `.o-road-gravel`, `.o-road-stone`, `.o-flow-arrow`).
- SVG overlays render as a sibling layer above the terrain fill, below
  the pin layer (if pins exist yet — they don't, slice 15).
- Reuse existing palette variables. New overlay colors:
  - River: `#5a87b8` (muted slate-blue, fits the parchment palette).
  - Road dirt: `#8b6f47` (medium tan).
  - Road gravel: `#7a7066` (warm gray).
  - Road stone: `#9c958a` (pale gray).
  - Flow arrow: `#5a87b8` (matches river).
- Stroke widths: rivers 3px, roads 2px (slightly subordinate).

### Edge index convention

The slice-9 hex rendering code already established edge index 0..5
mapping to specific physical edges. Use the existing convention. The
expected mapping is `0=NE, 1=E, 2=SE, 3=SW, 4=W, 5=NW` for pointy-top
hexes; if slice 9 rotated this, mirror the rotation rather than
reinvent it.

---

## Stage 1 — Schema migration, click-to-add, basic rendering

Goal: in paint-road or paint-river mode, click inside a hex and the
nearest edge gets added to that hex's overlay. Roads and rivers
render as basic lines on the map. Schema is v10. No drag, no erasers,
no detail editor, no flow arrows, no road colors yet.

### What to add

1. **Schema bump v9 → v10**:
   - Change `schemaVersion: 9` to `schemaVersion: 10` everywhere.
   - In `load()`, add `if (v < 10) migrateToV10(parsed);` to the
     migration chain.
   - Migration body:
     ```js
     function migrateToV10(parsed) {
       parsed.hexes = parsed.hexes || {};
       Object.values(parsed.hexes).forEach(h => {
         if (!Array.isArray(h.overlays)) h.overlays = [];
       });
     }
     ```
     This is purely defensive — no existing hexes have overlays in
     practice.

2. **Map toolbar modes**:
   - Add two new mode buttons next to the existing terrain paint mode:
     "🛤 Paint Roads" and "🌊 Paint Rivers".
   - Active mode visual: same indicator style as the existing terrain
     paint mode active state.
   - Clicking the active mode button again deactivates (returns to
     `mapMode: null`).
   - Switching to a paint mode deactivates terrain paint mode and
     vice versa — the modes are mutually exclusive.

3. **Click handler in paint modes**:
   - In `paintRoad` or `paintRiver` mode, clicking inside a hex:
     1. Compute the nearest edge to the click point. Use the hex
        center as the origin; the edge whose midpoint is closest to
        the click point wins. Tie-breaking by lower edge index.
     2. Find or create the overlay of that type on that hex:
        ```js
        let ov = hex.overlays.find(o => o.type === T);
        if (!ov) {
          ov = { type: T, edges: [], flowFrom: null };
          if (T === 'road') ov.surface = 'dirt';
          hex.overlays.push(ov);
        }
        ```
     3. If the nearest edge isn't already in `ov.edges`, append it.
        If it IS already there, do nothing (no toggle — that's eraser
        mode's job).
     4. `save()` and re-render the map.

4. **Rendering**:
   - In whatever function renders the hex grid, after the terrain fill
     and labels but before any pin layer, render overlays:
     ```js
     hex.overlays.forEach(ov => {
       const color = ov.type === 'river'
         ? '#5a87b8'
         : '#8b6f47';   // dirt is the only color in stage 1
       const width = ov.type === 'river' ? 3 : 2;
       ov.edges.forEach(eIdx => {
         const [mx, my] = edgeMidpoint(hex, eIdx);
         drawLine(centerX, centerY, mx, my, color, width);
       });
     });
     ```
     Use whatever line-drawing primitive is consistent with the rest
     of the map renderer (canvas `lineTo` / SVG `<line>` etc.).
   - `edgeMidpoint(hex, eIdx)` is a new helper if one doesn't exist.
     Compute it from the existing hex geometry math.

### Constraints

- No drag-trace yet — only single-click adds.
- No erasers — overlays can't be removed in stage 1. (You can
  manually clear via devtools if testing requires.)
- Roads always render in dirt color. No surface variants yet.
- No flow arrows on rivers.
- The painter does NOT cross hex boundaries on click — clicks only
  affect the hex they land in.

### Test checklist for Stage 1

- Reload after deploy. Existing hexes load cleanly with `overlays: []`
  defensively populated. Confirm `localStorage.getItem('wm_unified_v1')`
  parses to `schemaVersion: 10` and a sample hex shows `overlays: []`.
- Click "Paint Roads" — button shows active.
- Click inside a hex near its right edge — a road segment appears as
  a half-line from center to that edge midpoint.
- Click again near the bottom edge of the same hex — the road extends
  to two edges (a corner).
- Click a third edge — three-spoke pattern appears.
- Click an already-added edge — nothing happens (no error, no toggle).
- Switch to "Paint Rivers" — paint roads deactivates.
- Click a hex — blue river segment appears.
- Switch to terrain paint mode — both river/road paint modes
  deactivate.
- Click on a hex with no terrain (unknown) in paint-overlay mode —
  the overlay still gets added. Overlays don't require terrain.
- Refresh — all overlays persist.
- Open devtools, inspect a hex with overlays — confirm shape matches
  the v10 schema (`type`, `edges`, `flowFrom: null`, `surface: 'dirt'`
  for roads).

---

## Stage 2 — Drag-trace painting + erasers

Goal: dragging across hexes paints continuous overlays the way the
terrain painter does. Erase modes remove overlays.

### What to add

1. **Drag-trace logic** for paint modes:
   - On mousedown in a paint mode: record the hex under the cursor as
     `lastHex`, and add the nearest edge to the click point (same as
     stage 1 click behavior).
   - On mousemove while button held:
     - If the cursor is in the same hex as `lastHex`, do nothing.
     - If the cursor crosses into a new hex `currHex`:
       - Determine the SHARED edge between `lastHex` and `currHex`.
         This is the edge index `eL` of `lastHex` adjacent to `currHex`,
         and the corresponding edge index `eC` of `currHex` adjacent to
         `lastHex`. (For pointy-top hexes the relationship is
         symmetric: edge `n` of one corresponds to edge `(n+3) % 6` of
         the neighbor.)
       - Add `eL` to `lastHex`'s overlay (if not already present).
       - Add `eC` to `currHex`'s overlay (if not already present).
       - Update `lastHex = currHex`.
     - Save once at mouseup, not on every mousemove. Re-render on
       every mousemove for visual feedback.
   - On mouseup: save and clear `lastHex`.
   - If the cursor leaves the canvas while dragging: same as mouseup.

2. **Eraser modes**:
   - Two new toolbar buttons: "Erase Roads" and "Erase Rivers".
     Mutually exclusive with paint modes and with each other.
   - In erase mode, clicking inside a hex:
     - Find the overlay of the matching type on that hex.
     - Remove it from `hex.overlays` entirely (not edge-by-edge).
     - Save and re-render.
   - Drag-trace in erase mode does the same per-hex full removal as
     the cursor passes through hexes (so you can sweep an area clean).

3. **Cursor styling**:
   - In any paint or erase mode, the cursor over the canvas should be
     `crosshair`. This may already be true for terrain paint — match
     the existing pattern.

### Constraints

- Adjacency math must be correct. If you're unsure of the
  edge-numbering, write a small test in devtools first. The convention
  established in slice 9 is the source of truth.
- The shared-edge formula `eC = (eL + 3) % 6` ASSUMES pointy-top hex
  with the standard edge indexing. If slice 9 used a different
  convention, derive the formula from that.
- Dragging from inside a hex out of the canvas and back in should not
  produce stray edges. When the cursor re-enters, treat the first
  re-entered hex as a fresh `lastHex` (no shared-edge add).
- Erasers do NOT preserve flow direction or surface variants on
  partial removes — they wipe the whole overlay of that type.

### Test checklist for Stage 2

- Switch to "Paint Roads", drag in a straight line across 4 hexes.
  Each interior hex has 2 edges (entry + exit) added; the start and
  end hexes have 1 edge each (or 2 if you clicked first then dragged).
  Continuous road appears.
- Drag a curve. Hexes at bends have 2 non-collinear edges → the road
  visually "turns" at the center.
- Drag across an already-painted area. Existing edges aren't
  duplicated; the line stays clean.
- Switch to "Erase Roads", click a single road hex — gone.
- Drag across a road network in erase mode — sweeps clean.
- Erase Roads doesn't affect rivers in the same hex, and vice versa.
- Drag from the canvas off the edge and back — no stray edges added
  when re-entering.
- Refresh — all painted overlays persist correctly.

---

## Stage 3 — Per-hex detail editor + flow arrows + road surfaces

Goal: in the hex detail card (or wherever per-hex editing happens
today), expose precise overlay editing — checkboxes per edge, flow
direction, road surface. Render flow arrows on rivers and color
variations on roads.

### What to add

1. **Hex detail panel additions**:
   - Below the existing terrain controls, add an "Overlays" section
     with two subsections: Road and River.
   - **Road subsection**:
     - "Surface" radio buttons: Dirt / Gravel / Stone (default Dirt).
       Changing surface saves and re-renders.
     - 6 checkboxes labeled with directional names — `NE / E / SE /
       SW / W / NW` corresponding to edge indices 0..5. Toggling a
       checkbox adds/removes that edge from the road overlay.
     - If no edges are checked after a toggle, remove the road overlay
       entirely (don't leave an empty-edges overlay).
   - **River subsection**:
     - 6 directional edge checkboxes (same labels as road).
     - "Flow from" dropdown listing each currently-checked edge by
       directional name, plus a "— none —" option. The selected edge
       index becomes `flowFrom` directly. Changing edges that
       removes the current `flowFrom` from `edges` resets it to null.
     - If no edges are checked, remove the river overlay entirely.
   - Section is hidden / shows "(no overlays)" when both overlays are
     absent and the hex is in view-only state. In edit state, always
     show controls.

2. **Flow arrow rendering**:
   - In the renderer, when a river overlay has `flowFrom !== null`:
     - At the edge midpoint of edge `flowFrom`, draw a small
       arrowhead pointing INTO the hex (toward the center).
     - At every OTHER edge midpoint in `edges`, draw a small
       arrowhead pointing OUT of the hex (away from center).
     - Arrowhead size: ~6px, color matches the river.
   - When `flowFrom === null`, no arrows.

3. **Road surface coloring**:
   - In the renderer, road color is determined by `ov.surface`:
     - `dirt`: `#8b6f47`
     - `gravel`: `#7a7066`
     - `stone`: `#9c958a`
   - Default to dirt if `surface` is missing or unrecognized.

4. **Surface persistence on edit**:
   - When a road's edges change (via paint/erase/checkbox), `surface`
     is preserved.
   - When a road overlay is fully erased and a new one is created
     (paint mode), the new one defaults to `dirt`.

### Constraints

- The 6 edge labels in the detail editor are the directional names
  (NE/E/SE/SW/W/NW) matching slice 9's edge index 0..5 convention.
- The detail editor and the paint tool must produce identical data
  shapes. Don't fork the data model.
- Changing surface should NOT clear edges or flow direction.
- `flowFrom` is the edge index 0..5 directly. If a paint, erase, or
  detail-editor edit removes the source edge from `edges`, set
  `flowFrom = null` in the same operation.

### Test checklist for Stage 3

- Open a hex's detail panel. Overlays section appears.
- Toggle "Edge 1" under Road — road appears with one edge to that hex.
- Toggle "Edge 4" — road extends.
- Switch surface from Dirt → Stone — road repaints in pale gray.
- Refresh — surface persists.
- Toggle all checked edges off — road overlay removed entirely.
- Repeat for River. Set `flowFrom` via dropdown to one of the checked
  edges. Arrow appears at that edge pointing inward, others pointing
  outward.
- Uncheck the source edge for a river — `flowFrom` resets to null,
  arrows disappear.
- Paint a road via the toolbar, then open the hex's detail panel —
  the surface dropdown shows Dirt and the correct edges are checked.
  Change surface to Gravel via the dropdown — the painted road's
  color updates.
- Flow direction works on a 4-edge river (1 source, 3 outflows).

---

## Stage 4 — Polish, docs, cascade verification

Goal: clean up, document, verify nothing else broke.

### What to do

1. **Cascade verification**:
   - Confirm no other entity references overlays (they shouldn't —
     overlays are leaves attached to hexes).
   - Confirm "Clear hex" / "Set terrain to unknown" does NOT clear
     overlays. (Overlays are independent of terrain; a road can cross
     unknown terrain.) If the existing clear-hex logic touches
     `overlays`, remove that.
   - Confirm deleting a hex (if such a path exists) clears overlays
     with it. Currently no UI deletes hexes — they only get their
     terrain reset. Note in summary if any code path can orphan an
     overlay.

2. **`docs/data-model.md` updates**:
   - Bump every reference to v9 → v10 in the doc.
   - Update the `Overlay` block:
     ```js
     {
       type:     "river" | "road",
       edges:    [number],           // 1–6 indices in [0..5], all meeting at hex center
       flowFrom: number | null,      // edge index 0..5 (must be in `edges`); rivers only
       surface:  "dirt" | "gravel" | "stone",  // roads only; default "dirt"
     }
     ```
   - Add a brief note: "Edge indices: `0=NE, 1=E, 2=SE, 3=SW, 4=W,
     5=NW` for pointy-top hexes."
   - Add a migration note:
     "Migration v9 → v10: `migrateToV10()` ensures every hex has
     `overlays: []` defensively. The `edges` constraint relaxed from
     '1 or 2' to '1–6' (no data change required; old data with 1–2
     edges is valid in the new schema). New optional `surface` field
     added to road overlays; defaults to `'dirt'`. `flowFrom`
     semantics changed from 'array index into edges' to 'edge index
     0..5 directly' — but no existing rivers have flowFrom set, so
     no data migration is needed."

3. **`ROADMAP.md`**: move "Slice 14: Map overlays (rivers, roads)"
   from Planned to Done. Update the header "as of slice 13 complete"
   → "as of slice 14 complete".

4. **Code cleanup**:
   - No dead code from earlier stages.
   - No console.log left behind.
   - The toolbar button order makes sense: Terrain | Paint Road |
     Paint River | Erase Road | Erase River. (Or whatever order
     reads naturally — but consistent.)

### Test checklist for Stage 4

- Fresh reload. State loads cleanly. No console errors.
- All four overlay modes (paint road/river, erase road/river) plus
  terrain paint plus the detail editor produce identical data shapes.
- Doc diff: schema bumped, Overlay block updated, migration note
  added.
- ROADMAP diff: slice 14 in Done, header updated.
- All previous tab functionality still works (factions, npcs, events,
  etc.). Map tab still pans/zooms (if it does).
- Setting a hex's terrain to unknown after painting a road through it
  leaves the road intact.

### Completion summary to write at the end

- Total lines added to `index.html` (roughly).
- Any decisions you made on edge naming, drag-trace edge-cases, or
  rendering specifics.
- Confirmation that slice 14 is complete and ready for slice 14.5
  (which depends on overlays for the road bonus calculation).
