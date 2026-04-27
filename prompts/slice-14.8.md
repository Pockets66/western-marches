# Slice 14.8: Network-smoothed rendering, water overlays, multi-source rivers

## How to work this file

This slice has three stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation before starting the next stage.

Do NOT proceed past a stage without explicit confirmation. Do NOT
batch stages. Do NOT rewrite `index.html` wholesale — make surgical
edits only.

If a stage looks like it will exceed ~600 lines of additions, pause
partway through and check in before continuing.

## Slice context

Slice 14.7 introduced quadratic Bezier curves per spoke, but the
independent-bow-per-spoke approach produces visible discontinuities
at hex boundaries — adjacent hexes' curves don't share tangents at
their shared edge midpoints, so rivers especially look choppy.

This slice replaces the per-spoke curve drawing with **network
smoothing**: build a graph of all overlay segments of a given type,
find connected runs, and render each run as a single smooth Catmull-Rom
spline that flows through hex centers and edge midpoints continuously.
This eliminates the discontinuities at the cost of a more complex
rendering pass.

Three other features land in this slice because they're tightly
coupled to the rendering rewrite:

1. **Multi-source rivers.** Replace `flowFrom: number | null` with
   `flowFromEdges: [number]` so a river can have multiple inflow
   edges (rivers converging at a junction).
2. **Water overlays (ponds and lakes).** New `type: "water"` overlay
   for standalone ponds and lakes, settable in the detail editor.
   Independent of rivers — a hex can have either or both.
3. **Junction trunk overrides.** New `trunkPair: [number, number] |
   null` field on overlays. When set, the manual pair becomes the
   trunk through that junction; otherwise auto-detect picks it.

This is a **schema-changing slice**: v11 → v12. Backup
`wm_unified_v1` from localStorage before starting Stage 1.

The 14.7 code that's preserved: per-type perpendicular offsets,
bridge calc semantic, render order. Those weren't the problem.
The 14.7 code that's replaced: per-spoke Bezier drawing.

### In scope

- Schema v12 migration (multiple field changes).
- Network graph build + run finding + Catmull-Rom spline rendering
  for rivers, roads, and routes.
- Auto-detect trunk through junctions (longest run by edge count;
  for rivers with `flowFromEdges` set, source-to-longest-outflow).
- Manual trunk override via detail editor (3- and 4-edge junctions
  only; 5+ edge junctions stay auto-detect).
- Endpoint circles at network terminal nodes (matching overlay
  color, ~5px radius). Suppressed where rivers meet water.
- Multi-source river editor: replace single-select "Flow from"
  dropdown with multi-checkbox "Inflow edges."
- New `type: "water"` overlay for ponds and lakes.
- Detail editor: "Water feature" subsection (none / pond / lake).
- River spokes don't render in water-terrain hexes (but stay in the
  network graph for tangent continuity).
- Endpoint circles suppressed where rivers meet water terrain or
  water overlays.
- Flow arrow rendering uses spline tangents, not edge normals.
- Cache invalidation on overlay or terrain mutation.

### Out of scope

- Editing lake outline shape (lakes are deterministic-wobbly circles;
  no hand-editing).
- Trunk overrides for 5+ edge junctions (auto-detect only).
- Smoothing across the path-highlight gold preview (still uses
  whatever it does today).
- Road / route flow direction (only rivers have flow).
- Pin layer interactions (slice 15).
- Animations.

### Schema changes (v11 → v12)

**Generalize `flowFrom`** on river overlays:

```js
// Before (v11):
{ type: "river", edges: [...], flowFrom: number | null, ... }
// After (v12):
{ type: "river", edges: [...], flowFromEdges: [number], ... }
```

`flowFromEdges` lists which edges in `edges` are inflows. Edges in
`edges` but not in `flowFromEdges` are outflows. Empty array means
no flow direction set (legacy behavior).

**Add `trunkPair`** on river / road / route overlays:

```js
trunkPair: [number, number] | null
```

When set to two edge indices, the renderer treats those as the trunk
of any junction in this hex; remaining edges are branches. When null,
auto-detect picks the trunk. Both edges in `trunkPair` MUST be members
of `edges`, or the override is ignored at render time. (Don't enforce
at write time; let the renderer fall back gracefully if state drifts.)

**Add `type: "water"`** as a fourth overlay type:

```js
{
  type: "water",
  size: "pond" | "lake",
  edges: [],         // unused, always empty
  // flowFromEdges, trunkPair, surface — all unused
}
```

Water overlays have no edge connectivity. A hex can have at most one
water overlay. A hex can have a water overlay AND a river overlay
simultaneously (river ends in lake).

### Migration v11 → v12

```js
function migrateToV12(parsed) {
  parsed.hexes = parsed.hexes || {};
  Object.values(parsed.hexes).forEach(h => {
    if (!Array.isArray(h.overlays)) h.overlays = [];
    h.overlays.forEach(o => {
      // Generalize flowFrom -> flowFromEdges (rivers only)
      if (o.type === 'river') {
        if (!Array.isArray(o.flowFromEdges)) {
          if (typeof o.flowFrom === 'number') {
            o.flowFromEdges = [o.flowFrom];
          } else {
            o.flowFromEdges = [];
          }
        }
        delete o.flowFrom;
      }
      // Add trunkPair (river/road/route)
      if (o.type === 'river' || o.type === 'road' || o.type === 'route') {
        if (!('trunkPair' in o)) o.trunkPair = null;
      }
      // Water overlays don't exist yet, no migration needed for them
    });
  });
}
```

If a river had `flowFrom: null` in v11, the migration produces
`flowFromEdges: []`. Log a warning to console for any river with 2+
edges and `flowFromEdges: []` after migration so the user can audit
those hexes:

```js
console.warn('migrateToV12: river without flow info', hexKey, edges);
```

### State additions

No new persistent `state.ui` fields needed. The render cache is a
module-level variable, not in state.

### Naming / CSS conventions

- CSS class prefix for water styles: `o-water-pond`, `o-water-lake`.
- Reuse existing palette. New colors:
  - Endpoint circles: same as overlay color, slightly darker.
  - Pond / lake fill: `#5a87b8` (river blue), with `#3e6280` outline.
- Module-level cache variables prefixed with `_overlayCache` (e.g.
  `_overlayCacheRivers`, `_overlayCacheRoads`, `_overlayCacheRoutes`,
  `_overlayCacheVersion`).

---

## Stage 1 — Schema v12 + network graph + smoothed trunks + endpoint circles + water-terrain rendering rule

Goal: schema migrated, rivers/roads/routes render as smooth
Catmull-Rom splines instead of per-spoke Beziers, with cross-hex
continuity. Endpoint circles at terminal nodes. River spokes don't
draw in water-terrain hexes. No water overlay yet (Stage 2). No
trunk override UI yet (Stage 2). No flow arrow tangent fix yet
(Stage 3).

### What to add

1. **Schema bump v11 → v12**:
   - Change `schemaVersion: 11` → `schemaVersion: 12` everywhere.
   - In `load()`, add `if (v < 12) migrateToV12(parsed);` to chain.
   - Implement `migrateToV12` per the section above.
   - Update any reads of `overlay.flowFrom` to use
     `overlay.flowFromEdges` instead. The full audit list:
     - Detail editor "Flow from" dropdown (Stage 2 will rebuild this
       as multi-checkbox; for Stage 1, gracefully read the new field
       and pick `flowFromEdges[0] || null` to keep the dropdown
       working as a single-source UI temporarily).
     - Flow arrow rendering (Stage 3 will rewrite this for tangents;
       for Stage 1, render arrows at every edge in `flowFromEdges`
       pointing inward, every other edge pointing outward).
     - Anywhere else `flowFrom` appears in the codebase.

2. **Network graph build** for each spline-bearing overlay type
   (rivers, roads, routes — NOT water):

   ```js
   function buildOverlayGraph(type) {
     // Returns: { nodes: Map<nodeKey, Node>, runs: Run[] }
     // nodeKey format: "col,row|edgeIdx"
     // Internal connections (within hex): every pair of edges in
     //   one overlay's edges array is mutually connected.
     // External connections (across hexes): edge eA in hex A is the
     //   same node as edge eB in adjacent hex B, where the (hexA, eA)
     //   and (hexB, eB) reference the same world-space edge.
   }
   ```
   - Adjacency: edge `n` of one hex shares its midpoint with edge
     `(n+3) % 6` of the corresponding neighbor hex (verify with the
     same shared-edge formula slice 14 stage 2 used).
   - Walk all hexes; for each overlay of the given type, register
     all edges as nodes and link via internal + external connections.

3. **Find runs**:
   ```js
   function findRuns(graph) {
     // Returns: array of Run objects.
     // A Run is a maximal connected path through the graph from
     // one terminal/junction to another.
     // Rules:
     //   - A "junction" is any hex with 3+ edges of this overlay
     //     type. Junctions are run endpoints.
     //   - A "terminal" is a hex with 1 edge of this overlay type
     //     OR an edge with no neighbor continuation. Terminals are
     //     also run endpoints.
     //   - Hexes with exactly 2 edges of this type are pass-through
     //     and are NOT run endpoints.
     // Each Run has: { nodes: Node[], hexes: hexKey[] }
   }
   ```

4. **Auto-detect trunk at each junction**:
   - For rivers WITH non-empty `flowFromEdges`:
     - Trunk = the longest path starting from any inflow edge,
       walking through outflow edges. Other edges branch off.
   - For rivers WITHOUT `flowFromEdges` (empty array), and for
     roads / routes:
     - Trunk = the two edges that participate in the longest run
       through the junction.
   - Tie-breaker for rivers (option A from earlier discussion):
     when two outflow paths from a source have equal length, pick
     the one with smaller angular deviation from the inflow direction.
     "Angular deviation" = absolute angle between (incoming edge
     normal pointing inward) and (outgoing edge normal pointing
     outward). Smaller is straighter-through.
   - Tie-breaker for roads / routes: smallest angular separation
     between the two edges (the most "straight-through" pair).
   - These are auto-detected per render — not stored. Stage 2 adds
     the manual override that takes priority.

5. **Render each run as a Catmull-Rom spline**:
   ```js
   function catmullRom(points, segmentsPerKnot = 10) {
     // Standard centripetal Catmull-Rom spline through a list of
     // points. Returns a flat array of intermediate points.
     // segmentsPerKnot determines smoothness vs perf.
   }
   ```
   Implement the centripetal variant — it doesn't overshoot at
   sharp angles like uniform Catmull-Rom does.

   Build the spline waypoints for each run by alternating:
   - Edge midpoint (offset per type) at one end
   - Hex center
   - Edge midpoint (offset per type) — internal pass-through
   - Hex center
   - ... etc
   - Edge midpoint (offset per type) at the other end

   For the offsets, reuse `OVERLAY_OFFSET` and the per-type tangent
   offset logic from slice 14.7's `offsetEndpoint` helper.

6. **Skip water-terrain hexes during rendering, NOT during graph
   build**:
   - The graph must still include water-terrain hexes (so the spline
     waypoints flow through them and tangent continuity is preserved
     on either side).
   - In the rendering pass: for each spline segment between
     consecutive waypoints, draw the segment ONLY IF neither waypoint
     is the center of a water-terrain hex.
   - Implementation: walk the spline waypoints in pairs; for each
     pair, check `state.hexes[hex].terrain === 'water'` for the hexes
     each waypoint belongs to. If either is water, skip drawing
     that segment.

7. **Branch rendering at junctions**:
   - For a junction with N edges where the trunk uses 2, render the
     trunk as one continuous spline (extending into adjacent hexes
     as part of its run).
   - Each remaining (N-2) branches: a separate short spline from the
     hex center to the branch's edge midpoint.
   - The branch spline's starting tangent at the hex center is
     computed from the trunk spline's tangent at the same point. This
     gives clean Y-shapes instead of overlapping splines.
   - Specifically: if the trunk's tangent at the junction's hex
     center points along vector `T`, the branch's starting tangent
     should be a weighted average of `T` and the branch's edge-normal
     direction. Use `0.3 * T + 0.7 * branchNormal` as the starting
     tangent, then let the spline curve naturally to the branch's
     edge midpoint. This produces a smooth peel-off rather than a
     sharp departure.

8. **Endpoint circles**:
   - For each terminal node in each network (a node with no
     continuation across hex boundaries):
     - Draw a small filled circle at that node's offset edge
       midpoint.
     - Radius: 5px.
     - Fill: same as overlay color, ~20% darker.
     - Suppression rule for Stage 1: suppress the endpoint circle
       if the terminal hex has `terrain === 'water'`.
   - 1-edge stub overlays produce one endpoint circle at the open
     end.
   - 2-edge pass-throughs produce no endpoint circles (both ends
     have continuations).
   - Networks that end at a hex without continuation produce
     endpoint circles.

9. **Render cache**:
   - Module-level variables:
     ```js
     let _overlayCache = { rivers: null, roads: null, routes: null };
     let _overlayCacheKey = null;
     ```
   - Cache key is a hash of all hex overlays + terrains. Compute
     from `JSON.stringify(state.hexes)` if quick, or a more targeted
     hash if performance matters.
   - On every render, compare current key to cached key. If
     different, rebuild graphs/runs/splines and cache the result.
   - Invalidate cache (set key to null) on every overlay or terrain
     mutation. Add an `_invalidateOverlayCache()` helper and call it
     from every paint/erase/detail-editor handler.

10. **Replace slice 14.7's per-spoke Bezier code**:
    - Find `drawSpokeBezier` (or whatever it was named) and remove it.
    - Find the renderer's overlay-drawing loop and replace its
      contents with the new graph-based approach.
    - Verify the per-type offsets and render order from 14.7 are
      preserved: terrain → rivers → roads → routes → path-highlight.
    - The `OVERLAY_OFFSET` constants and offset endpoint helper from
      14.7 stay — they're reused.

### Constraints

- Catmull-Rom splines should use the centripetal parameterization
  to avoid overshoot. If implementing from scratch:
  ```js
  function catmullRomSegment(p0, p1, p2, p3, t) {
    // centripetal: alpha = 0.5
    // Use the standard centripetal formulation.
  }
  ```
- Spline rendering uses canvas `lineTo` along intermediate points,
  OR an SVG `<path>` with `d="M ... C ..."` cubic Bezier
  approximations. Match whatever the existing canvas/SVG approach is.
- Cross-hex continuity is achieved by sharing offset endpoints
  exactly: hex A's run ends at the same point hex B's run begins.
  This is automatic given the same `OVERLAY_OFFSET` rule applied on
  both sides — no special-case math needed.
- `flowFromEdges` reads must be defensive: if missing, default to `[]`.
- For Stage 1, the detail editor's "Flow from" UI keeps working as a
  single-edge selector (reads `flowFromEdges[0]` if non-empty;
  writes a single-element array). Stage 2 rebuilds this as a
  multi-checkbox. Don't break it in the meantime.

### Test checklist for Stage 1

- Reload after deploy. State migrates cleanly. Confirm
  `localStorage.getItem('wm_unified_v1')` parses to `schemaVersion:
  12`. Sample river overlay shows `flowFromEdges: []` (or the
  migrated single-element array) and `trunkPair: null`.
- All previously-painted overlays still render.
- Rivers crossing 5+ hexes appear as continuous, smooth lines with
  NO visible kinks at hex boundaries. This is the primary visual
  test — compare to slice 14.7 behavior.
- Roads similarly continuous.
- Routes (if any) similarly continuous.
- A 1-edge river renders with a small endpoint circle at the open end.
- A 2-edge pass-through river has no endpoint circles.
- A 3-edge river junction renders with the trunk auto-detected:
  - If `flowFromEdges` is set, the trunk is source → longest outflow.
  - If not set, the trunk is the two edges most opposite each other.
  - Branch peels off at a smooth angle.
- A 4-edge junction (X-shape): trunk auto-detected, 2 branches peel
  off cleanly.
- A river crossing a water-terrain hex: spokes render up to but not
  through the water hex. The river appears to enter the water on
  one side and emerge on the other, with the water terrain visually
  filling the middle.
- The river splines on either side of a water hex have matching
  tangents at the water hex's edges (no kinks).
- Endpoint circles do NOT appear where a river meets water terrain.
- Drag-paint a road through 5 hexes. After mouseup, render is
  smooth across all 5.
- Erase a road segment → cache invalidates → re-render shows
  updated network without stale segments.
- Edit a hex's overlays via the detail editor → cache invalidates,
  re-renders.
- Edit a hex's terrain to water → cache invalidates, river spokes
  through that hex disappear from rendering (graph still has them).
- Edit a hex's terrain back from water → river spokes return.
- Performance: a map with 50+ overlay hexes renders in <100ms on
  redraw with cache hit; <500ms on cache miss.

---

## Stage 2 — Junction trunk override UI + water overlays + multi-source river editor

Goal: detail editor gains controls for trunk override (3- and
4-edge junctions), water feature (none/pond/lake), and multi-source
river inflow. Water overlays render. Rivers ending in water overlays
clip to the water boundary.

### What to add

1. **Trunk override UI in detail editor**:
   - For each overlay type (river, road, route) where the hex has
     3 or 4 edges of that type: add a "Trunk pair" dropdown below
     the edge checkboxes.
   - Options:
     - "(auto)" → `trunkPair: null`
     - For each pair of edges (e.g. for edges [0, 2, 4]: "NE↔SE",
       "NE↔W", "SE↔W") → set `trunkPair` to that pair.
   - For 5+ edge junctions: don't show the dropdown. (If a previous
     `trunkPair` was set and the user later adds a 5th edge, the
     `trunkPair` becomes invisible but is still respected by the
     renderer — and still validated: if either edge is no longer
     in `edges`, fall back to auto-detect.)
   - Changing the dropdown saves and invalidates cache.

2. **Multi-source river editor**:
   - Replace the single-select "Flow from" dropdown in the river
     subsection with a multi-checkbox "Inflow edges" group.
   - Each currently-checked edge gets an inline checkbox: "Inflow?".
     Toggling adds/removes that edge index from `flowFromEdges`.
   - Outflow is implicit (any checked edge not in `flowFromEdges`).
   - When `edges` changes (a river edge is removed), filter
     `flowFromEdges` to remove any edges no longer in `edges`. Same
     defensive trim that `trunkPair` gets.

3. **Water overlay rendering**:
   - Render BEFORE rivers in the z-order. New stack:
     ```
     1. terrain fill
     2. water overlays (ponds and lakes)
     3. rivers
     4. roads
     5. routes
     6. path-highlight
     ```
     Wait — that's wrong. Water overlays should render AFTER rivers
     so the river's spline endpoints can be hidden under the water
     blob. Actually no: river splines need to terminate at the water
     boundary, not be drawn through it then covered. Let's render
     water FIRST (so we know its shape), then rivers terminate at
     the water boundary by clipping their spline endpoints.
     
     Final order:
     ```
     1. terrain fill
     2. water overlays  <- drawn first, but their bounding circle is
                          known and used for clipping
     3. rivers (with endpoint clipping to nearby water overlays)
     4. roads
     5. routes
     6. path-highlight
     ```

   - **Pond rendering**: a circle at the hex center.
     - Radius: ~25% of the hex apothem.
     - Fill: `#5a87b8`. Outline: `#3e6280`, 1.5px.
   - **Lake rendering**: a deterministic-wobbly circle at the hex
     center.
     - Base radius: ~50% of the hex apothem.
     - 8 evenly-spaced radial offsets, each varying ±15% of base
       radius based on `hashString(hexKey + ':lake')`.
     - Fill / outline: same as pond.
     - Render as an SVG `<path>` with cubic Beziers between offset
       points (or canvas `quadraticCurveTo` between them) so the
       outline is smooth, not jagged.

4. **River-into-water clipping**:
   - For each river spline endpoint that's in a hex with a water
     overlay (same hex), use the water overlay's bounding circle
     as the actual endpoint.
   - Bounding circle: pond uses radius 25% apothem; lake uses base
     radius 50% apothem (NOT the wobbled outline — the cheap
     bounding circle approach we discussed).
   - Move the spline's terminal waypoint from the offset edge
     midpoint to the intersection of the spline path with the water
     bounding circle. In practice: cut the spline short so it ends
     at the bounding circle's edge.
   - Suppress the endpoint circle for any spline ending inside a
     water overlay hex.

5. **Detail editor "Water feature" subsection**:
   - In each hex's detail panel, alongside Road and River
     subsections, add a "Water feature" subsection.
   - Radio buttons: "(none)" / "Pond" / "Lake".
   - Selecting Pond / Lake adds a `{ type: "water", size: "pond"|
     "lake", edges: [] }` overlay to the hex (or updates an existing
     one's `size`).
   - Selecting "(none)" removes any existing water overlay from the
     hex.
   - Saves and invalidates cache on every change.

6. **Cascade water overlay through deletes / clears**:
   - "Clear hex" (terrain reset) does NOT remove water overlays.
     Water is independent of terrain.
   - The existing erase-river / erase-road / erase-route modes do
     NOT touch water overlays.
   - Add nothing new here — water can only be removed via the
     detail editor's "(none)" option.

### Constraints

- A hex may have at most ONE water overlay.
- Water overlays don't appear in the network graph or runs (they're
  not connected to other overlays).
- Endpoint circles for rivers ending in water overlays: suppressed.
- River splines crossing through a hex that has a water overlay:
  the spline still passes through the hex center for tangent
  continuity, but is clipped at the water bounding circle. So a
  river that has a 2-edge pass-through in a lake hex would render
  as two separate splines: one entering and clipping at the lake,
  one exiting and clipping at the lake. The lake's blob fills the
  visible middle.
- For pure pass-through (2-edge river in lake hex), this looks like
  "river enters lake, lake, river exits lake" — natural.
- For a 3+ edge junction inside a lake: each branch terminates at
  the lake bounding circle. Trunk auto-detect / override still
  applies but visually all the spokes converge into the lake.

### Test checklist for Stage 2

- Open detail editor on a 3-edge road junction. "Trunk pair"
  dropdown appears with 3 options + "(auto)". Default is "(auto)".
  Pick a different pair → render updates immediately, the chosen
  pair becomes the trunk and the third edge peels off as a branch.
  Refresh → choice persists.
- Open detail editor on a 4-edge road junction → 6 options + "(auto)".
- Open detail editor on a 5-edge road junction → no trunk dropdown
  (just edge checkboxes).
- Set `trunkPair` on a 4-edge junction, then in the detail editor
  uncheck one of the trunk's edges → the renderer falls back to
  auto-detect (the override is invalid).
- Open detail editor for a river with 3+ edges. "Inflow edges"
  multi-checkbox shows one checkbox per checked edge. Toggle two
  to be inflows, leave one as outflow. Render shows the trunk
  going from one inflow through the outflow (the longest path),
  with the second inflow as a branch.
- A river with all-inflow edges and no outflow: renders as before
  (3 spokes converging) with no special pool. (Pools as a feature
  are out of scope; the auto-pool detection from earlier discussion
  was dropped.)
- Add a Pond to a hex via detail editor. Small blue circle appears
  at center.
- Add a Lake to the same hex (replacing the pond). Larger
  irregular blue blob appears.
- Set water back to "(none)". Blob disappears.
- Add a Pond to a hex that already has a 2-edge river. River
  spokes terminate at the pond's edge, not the hex center. No
  endpoint circles inside the pond.
- Add a Lake to a hex with a 4-edge river junction. All 4 spokes
  terminate at the lake's bounding circle. Lake blob fills the
  visible middle.
- Two adjacent hexes both with lakes: each lake renders independently.
  River between them (if any) terminates at each lake.
- Verify lake outline is deterministic: same hex always wobbles the
  same way across reloads.
- Confirm `state.hexes[hex].overlays` has `{ type: 'water', size:
  'lake', edges: [] }` after adding a lake.

---

## Stage 3 — Flow arrow tangents, polish, docs

Goal: flow arrows render along spline tangents (not edge normals)
so they visually flow with the river. Performance and cleanliness
pass. Docs updated.

### What to add

1. **Flow arrow rendering with spline tangents**:
   - For each river overlay, for each edge in `flowFromEdges`:
     - Find the spline waypoint corresponding to that edge midpoint.
     - Compute the spline's tangent at that point (the derivative
       of the Catmull-Rom segment).
     - Draw an arrowhead pointing INWARD along the tangent (toward
       the hex center).
   - For each edge in `edges` but NOT in `flowFromEdges`:
     - Same calculation, arrow pointing OUTWARD along the tangent.
   - Arrow size: 6px. Color: same as river, ~30% darker.
   - Suppress arrows on edges that have a continuation in the
     adjacent hex (i.e. only render arrows at terminal-network
     nodes). The user wants to see WHERE flow enters and exits the
     network, not at every internal segment.
   - Suppress arrows where the river meets a water overlay or water
     terrain (consistent with endpoint circle suppression).

2. **Performance check**:
   - Test a map with ≥50 overlay hexes.
   - Render with cache hit: <50ms.
   - Render with cache miss (after a paint stroke): <500ms.
   - If either fails, profile and optimize. Likely targets:
     - Hash function (use a faster hash than `JSON.stringify`).
     - Reduce `segmentsPerKnot` from 10 to 6 if splines look
       acceptable.

3. **Edge case audit**:
   - A river with 1 edge and `flowFromEdges: [thatEdge]`: renders
     as a stub spoke with an inward-pointing arrow at the edge
     and an endpoint circle at the open end where the spline
     terminates inside the hex (well, at the offset midpoint of
     the only edge). Actually: a 1-edge river HAS no inside
     endpoint — the spoke goes from hex center to edge midpoint.
     The endpoint is the edge itself. So the arrow at the edge is
     the only marker. No internal endpoint circle.
   - A river with 0 edges: shouldn't exist (the editor removes
     overlays with no edges). Verify the renderer doesn't crash if
     it does.
   - Cyclic networks (a river that loops back to itself): the run
     finder should handle gracefully — treat as a single closed
     run if both endpoints are the same node. Splines render
     normally; just don't put endpoint circles or arrows.
   - Disconnected networks: each connected component is its own
     set of runs.

4. **`docs/data-model.md` updates**:
   - Bump every reference to v11 → v12.
   - Update the Overlay block:
     ```js
     // River
     { type: "river", edges: [number], flowFromEdges: [number],
       trunkPair: [number, number] | null }
     // Road
     { type: "road", edges: [number], surface: "dirt" | "gravel" |
       "stone", trunkPair: [number, number] | null }
     // Route
     { type: "route", edges: [number], trunkPair: [number, number]
       | null }
     // Water (NEW)
     { type: "water", size: "pond" | "lake", edges: [] }
     ```
   - Add a section on rendering:
     ```
     Overlay rendering uses network smoothing: each overlay type's
     edges are connected into a graph, runs are extracted, and each
     run is rendered as a centripetal Catmull-Rom spline. Adjacent
     hexes' splines share offset endpoints exactly, eliminating
     visible kinks. At junctions (3+ edges), the trunk is either
     manually set (`trunkPair`) or auto-detected (longest run by
     edge count; for rivers with `flowFromEdges`, source-to-longest-
     outflow). Branches peel off the trunk with tangent matching.
     Endpoint circles mark terminal nodes, except where the network
     meets water (terrain or overlay). Water overlays clip incoming
     spline endpoints to their bounding circle.
     ```
   - Add the migration note.

5. **`ROADMAP.md`**: move "Slice 14.8: Network-smoothed rendering"
   from Planned to Done. Update the header.

### Constraints

- Spline tangent at an arbitrary point is the derivative of the
  Catmull-Rom segment that point belongs to. If implementing from
  scratch, use the analytical derivative; don't sample with finite
  differences (less accurate).
- Arrow rendering must not double up: an arrow on an edge with
  continuation across hexes should NOT render (it's a network
  internal point, not a terminal).
- Cache must invalidate on terrain changes (because water terrain
  affects spline rendering).

### Test checklist for Stage 3

- A river with `flowFromEdges: [0]` and `edges: [0, 3]` renders an
  inward arrow at edge 0 and an outward arrow at edge 3 — both
  arrows point along the spline tangent, NOT along the edge normal.
  For a straight pass-through they'll be similar; for a curved run
  the difference is visible.
- A river with `flowFromEdges: [0, 2]` and `edges: [0, 2, 4]`
  renders inward arrows at 0 and 2, outward at 4. All along
  spline tangents.
- Internal network nodes (where the river continues into the next
  hex) have no arrows.
- Performance: smooth render on a busy map.
- Schema doc reflects v12, all four overlay types documented.
- Migration note documents the v11 → v12 changes.
- ROADMAP shows slice 14.8 in Done.
- All previous tabs still work.
- Slice 14.6 undo still works (paint a road, undo, etc.) — the new
  rendering doesn't break the undo system.
- Travel calc still works correctly (slice 14.5's bridge semantic
  preserved).

### Completion summary

- Total lines added/changed in `index.html`.
- Note any decisions on spline parameters (segmentsPerKnot, bow,
  etc.) you tuned differently.
- Confirmation that slice 14.8 is complete.
- Note any visual issues that emerged that might warrant a polish
  follow-up.
