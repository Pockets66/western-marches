# Slice 14.9: Center-origin support for overlays

## How to work this file

This slice has two stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation before starting the next stage.

## Slice context

Today, every overlay (river, road, route) must connect to one or
more hex edges. There's no way to express "this hex is the source
of the river" — only "water flows in from edge X." Same for roads
("the King's Road begins at the castle hex") and routes ("the
party set out from this hex").

This slice adds a "center origin" concept: an overlay can be
anchored at the hex center and extend outward to one or more edges,
or — at the limit — exist as a single point at the hex center with
no edges at all.

This is a **schema-changing slice**: v12 → v13. Backup
`wm_unified_v1` from localStorage before starting Stage 1.

### In scope

- Schema v13 migration (defensive: `originAtCenter: false` on every
  overlay that doesn't have it).
- Detail editor: new "Origin at center" checkbox in each overlay's
  subsection (river, road, route).
- Renderer: when `originAtCenter` is true, treat the hex center as
  a network terminal (an endpoint of the network, like a leaf node).
- Endpoint circle: a slightly larger / styled circle at the hex
  center when origin is set, distinguishing "spring source" from
  edge endpoints.
- For rivers: `originAtCenter: true` implies water flows OUT from
  the center to all edges (every edge is an outflow). This
  interacts with `flowFromEdges` — see "Flow semantics" below.
- For roads / routes: `originAtCenter: true` is purely directional/
  visual; no flow concept involved.
- Edge case: an overlay can have `originAtCenter: true` AND zero
  edges. This is a "lone marker" — a road that begins and ends in
  the same hex (a town with no roads in or out yet), a river spring
  that hasn't reached any edge yet, etc.

### Out of scope

- Multiple origin points in the same hex (one origin per overlay
  type maximum).
- Special "spring" or "terminus" graphics beyond the existing
  endpoint circle (just a marker; no fancy icon).
- Origin-at-center for water overlays (water already represents a
  hex-center feature; the concept doesn't apply).
- Trunk/branch logic changes — the trunk auto-detect just treats
  the center as one endpoint of the longest run when present.

### Schema changes (v12 → v13)

Add `originAtCenter: boolean` to river / road / route overlays:

```js
// River
{ type: "river", edges: [number], flowFromEdges: [number],
  trunkPair: [number, number] | null,
  originAtCenter: boolean }              // NEW
// Road
{ type: "road", edges: [number], surface: "dirt"|"gravel"|"stone",
  trunkPair: [number, number] | null,
  originAtCenter: boolean }              // NEW
// Route
{ type: "route", edges: [number],
  trunkPair: [number, number] | null,
  originAtCenter: boolean }              // NEW
// Water (unchanged — no center-origin concept for water)
{ type: "water", size: "pond"|"lake", edges: [] }
```

### Migration v12 → v13

```js
function migrateToV13(parsed) {
  parsed.hexes = parsed.hexes || {};
  Object.values(parsed.hexes).forEach(h => {
    (h.overlays || []).forEach(o => {
      if (o.type === 'river' || o.type === 'road' || o.type === 'route') {
        if (!('originAtCenter' in o)) o.originAtCenter = false;
      }
    });
  });
}
```

### Flow semantics for rivers with center origin

A river with `originAtCenter: true` has the hex center as an
implicit "source." Water flows from the center outward to all
edges in `edges`. This means:

- If `originAtCenter: true`, every edge in `edges` is implicitly
  an outflow.
- `flowFromEdges` should be `[]` (no edges are inflow). If a user
  sets `originAtCenter: true` while `flowFromEdges` is non-empty,
  warn but allow it — the renderer treats `flowFromEdges` entries
  as inflow regardless (so a river could have center-spring AND
  an additional inflow from one edge, which is geologically
  bizarre but not invalid).
- Detail editor enforces: when "Origin at center" is checked,
  the inflow checkboxes for individual edges become disabled
  (visually communicating that center-origin overrides them).
  Toggling off "Origin at center" re-enables them.

### Network graph integration

The hex center, when `originAtCenter: true`, becomes a node in the
network graph alongside the edge nodes. It connects to every edge
in the same overlay's `edges` array (just like edge nodes connect
to each other within a hex through the center).

The graph build needs one additional rule: for each overlay with
`originAtCenter: true`, add a "center" node identified as
`"col,row|center"` (instead of `"col,row|edgeIdx"`). It connects
internally to every edge node in the same overlay. It has no
external connections (the center never connects across hex
boundaries).

A run that includes a center node has the center as one of its
endpoints. The renderer treats it as a terminal — endpoint circle
goes there.

A 0-edge overlay with `originAtCenter: true` is a degenerate
network: one node (the center), no connections, one run of length
1. The renderer just draws an endpoint circle at the center.
Nothing else.

---

## Stage 1 — Schema v13 + center-origin renderer + endpoint marker

Goal: schema migrated, the renderer respects `originAtCenter` for
rivers, roads, and routes. Center origins render as endpoint
markers (slightly larger / different style than edge endpoints).
No detail editor UI yet (Stage 2).

### What to add

1. **Schema bump v12 → v13**:
   - Change `schemaVersion: 12` → `schemaVersion: 13`.
   - In `load()`, add `if (v < 13) migrateToV13(parsed);`.
   - Implement migration per the section above.

2. **Network graph builder updated**:
   - For each overlay with `originAtCenter: true`, add a center
     node to the graph keyed `"hexKey|center"`.
   - The center node connects internally to every edge node in the
     same overlay.
   - Center nodes have no cross-hex external connections.

3. **Run finder updated**:
   - Center nodes are network terminals. A run can have a center
     node as one of its endpoints.
   - A 0-edge overlay with `originAtCenter: true` produces a
     degenerate run with a single center node and no segments to
     draw. The renderer should handle this (draw the endpoint marker,
     skip the spline).

4. **Spline rendering with center-origin runs**:
   - A run that starts/ends at a center node has its first/last
     waypoint at the hex center (no edge midpoint offset).
   - The Catmull-Rom spline naturally handles this — it's just one
     more waypoint.
   - For a 1-edge overlay with `originAtCenter: true`, the run is
     center → edge midpoint. Render as a short spline (essentially
     a curved spoke from center to the offset edge midpoint).

5. **Endpoint markers**:
   - Edge endpoints already render as 5px-radius filled circles
     (from slice 14.8 Stage 1).
   - Center origins render as 7px-radius filled circles at the hex
     center, with a 1.5px outline in the overlay's color.
     Distinguishes "spring source / road origin" from "river ends /
     road dead-end."
   - Suppression rule: same as edge endpoints — suppress where the
     center is inside a water overlay (the water absorbs it).

6. **Cache invalidation**:
   - The existing cache invalidation should already cover changes
     to overlay structure. Verify that toggling `originAtCenter`
     via direct state mutation (test by manually setting it in
     devtools) triggers a re-render.

### Constraints

- The Stage 1 work is renderer-only. No editor UI changes yet.
- Test by manually setting `originAtCenter: true` on existing
  overlays via devtools console. The detail editor doesn't yet
  expose this — that's Stage 2.
- Center-origin marker styling differs from edge-endpoint styling
  to make the source visually distinct.

### Test checklist for Stage 1

- Reload after deploy. State migrates to v13. Verify
  `localStorage.getItem('wm_unified_v1')` parses to
  `schemaVersion: 13`. Sample river overlay shows
  `originAtCenter: false` (defensive default).
- Existing overlays render exactly as before (since
  `originAtCenter: false` for all of them).
- Devtools test: set `originAtCenter: true` on a 1-edge river
  overlay (e.g. `state.hexes['3,3'].overlays[0].originAtCenter =
  true; renderMap();`). The river now renders with a 7px circle at
  the hex center connected to the edge midpoint by a spline. The
  edge end has its normal 5px endpoint circle.
- Devtools test: `originAtCenter: true` on a 3-edge river. All 3
  spokes converge at the hex center, which has a 7px endpoint
  marker. Each edge end has its 5px endpoint circle (where the
  edges are terminal in the network).
- Devtools test: `originAtCenter: true` and `edges: []`. Just a
  7px endpoint marker at the hex center, no spokes.
- Devtools test: `originAtCenter: true` on a road. Same behavior
  but road color.
- Suppression: center-origin in a hex with a water overlay → no
  marker (water absorbs it). Spline still terminates at the water
  bounding circle.
- All previous overlay rendering still works. Slice 14.8 features
  unaffected (junction trunks, multi-source rivers, water overlays,
  bridge calc, undo, etc.).

---

## Stage 2 — Detail editor UI + flow integration + docs

Goal: the detail editor exposes `originAtCenter` as a checkbox.
Inflow editing for rivers correctly disables when center-origin is
set. Docs reflect v13.

### What to add

1. **Detail editor checkbox**:
   - In each overlay subsection (river, road, route), add a checkbox
     labeled "Origin at center" below the edge checkboxes and above
     the trunk-pair dropdown.
   - Toggling it sets/clears `overlay.originAtCenter`, saves, and
     invalidates the cache (which triggers a re-render).
   - The checkbox is visible only when at least one edge is checked
     OR the overlay is a "lone marker" (0 edges + originAtCenter).
     Specifically: don't show the checkbox if the hex has no
     overlay of this type at all. To create a lone-marker overlay
     (0 edges, just a center), the user must first toggle a temporary
     edge on, then enable origin, then untoggle the edge. That's a
     fine workflow — don't add a special "create lone marker" button.

   Wait — that workflow is annoying. Let me revise: the checkbox is
   available in the overlay subsection whenever the user has
   interacted with the subsection at all. If toggling "Origin at
   center" creates an overlay where one didn't exist, that's fine
   — initialize with `edges: []`, `originAtCenter: true`, and
   sensible defaults for other fields.

2. **Inflow editor integration (rivers only)**:
   - When `originAtCenter` is true on a river, the per-edge "Inflow?"
     checkboxes (from slice 14.8 Stage 2's multi-source editor) are
     disabled and visually grayed out.
   - A help text appears: "Center origin: water flows out from the
     center to all edges." (Or similar.)
   - When "Origin at center" is toggled OFF, the inflow checkboxes
     re-enable.
   - Migration safety: if a state somehow has both
     `originAtCenter: true` AND `flowFromEdges: [0, 2]`, the
     renderer renders the center as a source AND edges 0 and 2 as
     inflows. The user can clean this up via the editor.

3. **Trunk-pair integration**:
   - A center-origin overlay with 3+ edges has the center as one
     terminal of the network. Trunk auto-detection treats the center
     as a regular endpoint. The trunk pair dropdown still applies to
     edge pairs only — the center isn't part of the dropdown options.
   - This works automatically if `trunkPair` is constrained to edge
     indices and the renderer's branch logic handles a center
     terminal as just another endpoint.

4. **Lone marker UI clarity**:
   - If the user has an overlay with `originAtCenter: true` AND
     zero edges, the subsection should still show its full controls
     (the "Origin at center" checkbox stays on; the surface dropdown
     for roads stays available). If they toggle off "Origin at
     center" while edges is empty, the overlay is removed entirely
     (consistent with "empty overlay = no overlay" rule).

5. **`docs/data-model.md` updates**:
   - Bump v12 → v13 throughout.
   - Add `originAtCenter` to the river / road / route overlay shape
     blocks.
   - Add a "center origin" section explaining:
     - The semantic: hex center as a network terminal.
     - For rivers: implicit outflow to all edges; `flowFromEdges`
       still works for additional inflow but is unusual to combine.
     - For roads/routes: purely visual / directional anchor.
     - Endpoint marker styling: 7px circle vs 5px for edge
       endpoints.
   - Add the migration note: "Migration v12 → v13: defensive
     `originAtCenter: false` added to all river / road / route
     overlays. No data change for existing maps."

6. **`ROADMAP.md`**: move "Slice 14.9: Center-origin support" from
   Planned to Done. Update the header.

### Constraints

- The "Origin at center" checkbox state is independent per overlay
  (a hex's road can have origin while its river doesn't).
- Toggling "Origin at center" doesn't auto-modify `edges` or
  `flowFromEdges`. The user manages those independently.
- Don't add an "Origin at center" checkbox to the water subsection
  (water has no center-origin concept).

### Test checklist for Stage 2

- Detail editor on a hex with a 2-edge river: "Origin at center"
  checkbox visible, default unchecked.
- Toggle "Origin at center" on. Renderer adds a 7px center marker;
  the river now appears to flow from center to both edges.
- Inflow checkboxes for the 2 edges are now disabled and grayed.
  Help text appears.
- Toggle "Origin at center" off. Inflow checkboxes re-enable. Help
  text disappears. Center marker disappears.
- Set "Origin at center" on a hex with no road overlay. New road
  overlay created with `edges: []`, `originAtCenter: true`,
  `surface: 'dirt'`. Renders as 7px tan circle at hex center.
- Toggle off "Origin at center" on a 0-edge overlay. Overlay
  removed entirely. (Verify in devtools that
  `state.hexes[hex].overlays` no longer includes it.)
- 3-edge river junction with "Origin at center" toggled on:
  trunk auto-detect picks two edges as the trunk; the center is
  treated as the run's other terminal.
- Refresh — all "Origin at center" toggles persist.
- `docs/data-model.md` shows v13 throughout, `originAtCenter` in
  all three overlay shape blocks, center-origin section explains
  semantics.
- `ROADMAP.md` shows slice 14.9 in Done.
- Slices 14.5, 14.6, 14.7, 14.8, 14.8.1 features all still work.

### Completion summary

- Total lines added/changed in `index.html`.
- Note any decisions about lone-marker UX or center-origin
  rendering details.
- Confirmation that slice 14.9 is complete.
