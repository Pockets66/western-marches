# Slice 14.10.1 — Overlay rendering & paint fixes

Pure render/interaction fixes on top of slice 14.10. **No schema change**, no
migration, no version bump. Touches `index.html` only.

## Motivation

Four visible bugs in the segment-based overlay system landed in 14.10:

1. **Cross-hex kinks.** Curved runs that pass through a hex boundary often hit
   an extreme angle right at the shared edge instead of bending smoothly into
   the next hex.
2. **Center-snapping squiggles.** Drag-painted roads and rivers tend to bounce
   between edge midpoints and the hex center, producing a path that dips to
   center on every hex along the trace.
3. **Ponds erase rivers.** Adding a pond to a hex that already has a river
   visually eliminates the river inside that hex (and sometimes adjacent hexes).
   Painting the river *after* the pond sometimes works, sometimes doesn't.
4. **Flow arrows at the wrong place.** River flow arrows render at the
   network's terminal endpoints rather than within each directed segment.

All four are in the rendering pipeline (`_renderNetworkType`,
`buildOverlayGraph`, `catmullRomPts`) or the drag-paint handler.

## Stages

Each stage is one logical fix. Stop after each stage for a "go" confirmation
before proceeding.

---

### Stage 1 — Ponds no longer erase rivers

**Symptom.** Hex with `water` overlay (pond or lake) + `river` overlay: the
river inside that hex disappears entirely. Sometimes the issue propagates into
adjacent hexes (the river vanishes one hex past the pond on either side).

**Root cause.** Two behaviors in `_renderNetworkType` compound:

- `nodeWp(k)` yanks **every** edge anchor in a water-overlay hex inward to the
  water-circle radius (25% apothem for ponds, 50% for lakes). That pulls the
  shared-edge anchor in this hex away from the matching anchor in the neighbor
  hex, breaking cross-hex continuity.
- The sub-path splitter treats any `C` node in a water-overlay hex as a hard
  cut, pushing a fresh empty sub-path. A `{e1,C}+{C,e2}` river through a pond
  hex therefore becomes two single-point sub-paths, both of which fail the
  `sub.length < 2` guard and render nothing.

**Fix.** In `_renderNetworkType`:

- Drop the water-overlay branch from `nodeWp`. Edge anchors stay at their
  normal offset position regardless of the hex's water overlay. (Keep the
  function — `nodeWp` is still called, just with no water-clipping logic.)
- In the sub-path splitter, only split on **water terrain** (`isWaterTerrain`),
  not on water overlays (`waterOvOf`). Z-order already handles the visual
  overlap: ponds and lakes paint on top of rivers.

**Test.**
- Paint a 5-hex straight river. Add a pond to the middle hex. River must
  remain visually continuous across all 5 hexes; the pond circle paints over
  the centermost portion of the river inside its hex.
- Same with a lake. Same with a `{e1, e2}` pass-through river. Same with a
  `{C, e1}` stub river ending in the pond hex.
- Hex with `water` *terrain* (not overlay) still splits the path — verify a
  river running into a water-terrain hex still terminates cleanly.

---

### Stage 2 — Smooth curves at hex boundaries

**Symptom.** A river or road traced across multiple hexes shows a visible kink
or cusp right at each shared edge, instead of curving smoothly through.

**Root cause.** `buildOverlayGraph` represents each shared edge as two graph
nodes (one per hex) joined by an external continuity edge. Both nodes resolve
to the **same world position** because the offset sign is globally consistent.
Run extraction therefore produces a waypoint list with consecutive duplicate
points. `catmullRomPts` cannot compute a meaningful tangent at a duplicated
point, producing a degenerate spline segment that visibly kinks.

A secondary issue: `catmullRomPts` is uniform-parameter Catmull-Rom. Centripetal
parameterization (alpha=0.5) is much more forgiving of irregular spacing, which
hex-network runs always have (short cross-edge hops alternating with longer
center-to-edge hops).

**Fix.**

- In `_renderNetworkType`, before splitting into sub-paths, dedupe consecutive
  waypoints whose `(x, y)` differ by less than `1e-6`. Keep the dedupe local to
  the spline input — don't mutate the run.
- Replace the uniform Catmull-Rom in `catmullRomPts` with centripetal
  Catmull-Rom: parameterize each segment by `dt = sqrt(distance)` instead of
  uniform `t ∈ [0, 1]`, and evaluate the spline using those non-uniform
  parameters. Phantom endpoints follow the same convention. (The data-model.md
  comment already claims centripetal — this brings the implementation in line.)

**Test.**
- Trace a long curving river that crosses 4–5 hex boundaries. No visible kinks
  at the shared edges; curvature continuous across boundaries.
- Pure pass-through: paint river `{NW, SE}` in a hex with neighbors that
  continue the run. Boundary curves smooth.
- Single-segment stubs (`{C, edge}`) still render correctly with no neighbor.
- Sharp 60° turns at edges (e.g., E→NE in one hex, neighbor with NW→W) bend
  visibly but without cusps.

---

### Stage 3 — Stop snapping to center on drag-paint

**Symptom.** Tracing a road or river across several hexes by drag produces a
squiggly path that dips to the center of most hexes, rather than a smooth line.

**Root cause.** Two compounding issues:

- `_nearestAnchor` uses Voronoi-style nearest-of-7 selection. The `C` cell
  occupies everything within ~half an apothem of the hex center, which the
  cursor inevitably crosses when dragging from one edge to the opposite edge.
  That produces `{enter, C}` + `{C, exit}` segments instead of `{enter, exit}`.
- The data is technically valid, but the renderer faithfully draws the C dip
  even though the run has graph degree 2 at that C node (i.e. C is acting as a
  pure pass-through, not a real junction).

**Fix.** Both sides:

- **Render side.** In `_renderNetworkType`, after run extraction and before
  splining, walk each run's interior nodes and drop any `C` node whose graph
  degree is exactly 2. That converts a `{e1, C} + {C, e2}` chain into a smooth
  `{e1, e2}` waypoint pair without changing the underlying segments. Junctions
  (deg ≥ 3) at C still keep the C waypoint and render as designed.
- **Paint side.** Add a new helper `_nearestDragAnchor(col, row, px, py)` that
  is identical to `_nearestAnchor` except it excludes `C` from the candidate
  set. Use it in the `mousemove` drag handler for both `paintRoad` and
  `paintRiver`. Click-paint (the `_finishOverlayDrag` stub-creation path) and
  the segment editor continue to use `_nearestAnchor` / `_nearestEdgeAnchor`
  unchanged — clicks still produce stubs, edits still allow C.

**Test.**
- Drag a road across 5 hexes from west to east. Result: 5 `{W, E}`
  pass-through segments, smooth straight line, no center dips.
- Drag a road from one hex's center *outward* (start near C, drag to an edge):
  starting anchor is whatever edge is closest at mousedown — verify clicks at
  C still produce a `{C, edge}` stub via the click-paint path.
- Existing data with explicit `{C, edge}` segments still renders the C
  correctly when C is a real junction (deg ≥ 3) — paint a Y-junction with
  three `{C, edge}` segments via the segment editor; all three branches meet
  at C.
- Existing data with a single isolated `{C, edge}` stub still renders the stub
  (deg-2 collapse only fires when C has exactly 2 graph neighbors).

---

### Stage 4 — Flow arrows mid-segment, not at terminals

**Symptom.** Flow arrows on rivers render at the network's degree-1 terminal
nodes. For a long river, that means one arrow at the source and one at the
mouth — interior segments with their own `flow` value get no arrow at all.

**Root cause.** The flow-arrow loop in `_renderNetworkType` walks `runs` and
draws only at terminals (`deg(k) === 1`).

**Fix.** Replace the terminals-only walk with a per-segment loop:

- For each `graph.edges` entry where `e.segment` is non-null and
  `e.segment.flow != null`:
  - Look up the two waypoint positions for `e.a` and `e.b` (apply the same
    `nodeWp` resolution used for the spline).
  - Compute `mx, my` = midpoint of those two positions.
  - Compute direction `dx, dy` from `from` to `to`. If `flow === 'toFrom'`,
    negate.
  - Skip if either endpoint is in a water-terrain hex (existing rule).
  - Draw the arrowhead at `(mx, my)` pointing along `(dx, dy)`.

The midpoint of the two anchor positions (as opposed to sampling the spline
curve) is fine for the gentle curvatures involved and keeps the math simple.

**Test.**
- 3-segment river `A→B→C→D` with `flow: 'fromTo'` on each segment: three
  arrows visible, one per segment, each in the middle of its segment.
- Mixed flow: middle segment `'toFrom'`, others `'fromTo'`: middle arrow
  points opposite direction.
- River with one `flow: null` segment (after Sync flow but before manual fix):
  that segment has no arrow; others do.
- Junction (Y-shape): each of the three branches has its own segment-level
  arrow; no special-case at C.
- Stub `{C, edge}` with flow set: arrow at midpoint between C and edge anchor.

---

## Out of scope

- Any schema change (no migration, schemaVersion stays at 13).
- Map undo (still slice 14.11).
- Any other rendering polish (line joins, dash phase, surface-color blending,
  lake-blocks-roads behavior, route styling).
- Touching the segment editor UI.

## Files

- `index.html` — all changes.
- `ROADMAP.md` — append slice 14.10.1 to Done section after final stage merges.
- `data-model.md` — add a one-line note in the rendering paragraph if the
  centripetal claim was previously aspirational; otherwise no change.

## Definition of done

- All four bugs above no longer reproduce on a fresh test map.
- No regressions in existing 14.10 behavior:
  - Y-junctions at C still render as Y, not as a smooth pass-through.
  - Lake-blocks-roads modifier still applies correctly.
  - Sync flow propagation unchanged.
  - Segment editor unchanged.
  - Travel calculator and route overlays unchanged.
- localStorage data created under 14.10 still loads cleanly (no schema delta
  to break anything, but verify).
