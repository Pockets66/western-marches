# Slice 14.10: Segment-based overlay model (consolidation)

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

Do NOT proceed past a stage without explicit confirmation. Do NOT
batch stages.

If a stage looks like it will exceed ~600 lines of additions, pause
partway through and check in before continuing. Stages 2 and 3 in
particular are at risk of running long.

## Slice context — read this section carefully

**This is a consolidation slice.** Slices 14 through 14.9 were the
exploratory phase of the overlay system. They surfaced limitations
that can't be patched cleanly in the current model:

- The edge-based representation (`edges: [number]`) forces all
  edges in a single hex's overlay to connect through the hex
  center. There's no way to express two parallel non-connecting
  roads in the same hex.
- Sharp turns are unavoidable because every spline waypoint passes
  through the hex center, even for simple two-edge curves.
- `flowFromEdges` and `originAtCenter` are awkward bolt-ons; they
  exist because the edge-based model has no first-class concept of
  "where does this segment of overlay actually go."
- Lake/road interaction is poorly defined — there's no clean way to
  say "the road must go around the lake" because the model doesn't
  represent the road's actual path through the hex.

This slice replaces the edge-based model with a **segment-based
model**: each overlay carries a list of segments, where each segment
is a line between two of the hex's seven anchor points (six edges +
one center). This naturally supports parallel paths, smooth turns
that bypass the center, and unambiguous lake-blocking.

**There is no real overlay data to migrate** — the user has no
deployed rivers/roads/routes/water in their saved state. The
migration code is therefore defensive only (set defaults, ensure
shape correctness). Old `edges`/`flowFromEdges`/`originAtCenter`/
`trunkPair` fields will be dropped from the schema.

**Subsumed slices:**
- Slice 14.9 (center-origin support) is dropped — center origins
  are naturally expressed as `from: 'C'` in a segment.
- Slice 14.8.2 (deferred patches: don't-delete-roads, z-order, lake
  gaps, sharp turns) is dropped — all four issues are fixed in this
  slice.

**Deferred to slice 14.11:**
- The map undo system (originally planned as slice 14.6) — building
  it on top of the segment model is smaller than building it on the
  edge model and porting.

**Preserved features (not changing):**
- Travel calculator (slice 14.5).
- Per-type perpendicular offsets (rivers/roads/routes don't overlap
  on shared anchor points).
- Network smoothing via Catmull-Rom splines.
- Manual junction trunk override concept (becomes implicit in the
  segment model — see Stage 2).
- Multi-source rivers.
- Water overlays (ponds, lakes).
- Continuity-snap in the painter.
- Single-edge erase via Shift modifier.
- Bridge calc semantic for travel.

This is a **schema-changing slice**: v12 → v13. Backup
`wm_unified_v1` from localStorage before starting Stage 1, even
though the migration is essentially a no-op for your current data.

### The model

**Hex anchor points.** Every hex has 7 anchor points:
- 6 edge midpoints (referred to by edge index 0–5)
- 1 center (referred to by sentinel `'C'`)

Anchor points are placed deterministically per hex. Adjacent hexes'
edge anchor points coincide exactly at the same world-space position
(this is what gives cross-hex continuity).

**Segment.** A segment is a connection between two distinct anchor
points within a single hex's overlay:

```js
{
  from: 'C' | 0 | 1 | 2 | 3 | 4 | 5,
  to:   'C' | 0 | 1 | 2 | 3 | 4 | 5,
  flow: 'fromTo' | 'toFrom' | null    // rivers only; ignored on others
}
```

Constraints:
- `from` and `to` must differ (no self-loops).
- A given anchor point can appear in multiple segments (that's how
  junctions work: three segments all meeting at `'C'`).
- Two segments with the same `{from, to}` pair (or reversed) within
  the same overlay are duplicates — the painter and detail editor
  should prevent this.

**Overlay shapes.**

```js
// River
{
  type: "river",
  segments: [{ from, to, flow }, ...]
}

// Road
{
  type: "road",
  surface: "dirt" | "gravel" | "stone",
  segments: [{ from, to }, ...]      // flow ignored if present
}

// Route
{
  type: "route",
  segments: [{ from, to }, ...]
}

// Water (unchanged from v12)
{
  type: "water",
  size: "pond" | "lake",
  segments: []                       // always empty; kept for shape uniformity
}
```

**Removed fields.** The following v12 fields no longer exist:
- `edges` (replaced by segments)
- `flowFromEdges` (replaced by per-segment `flow`)
- `trunkPair` (implicit: a junction's trunk is whichever segment
  doesn't include `'C'`, OR the longest run if multiple non-center
  segments exist; auto-detected at render time, no manual override
  in this slice)
- `originAtCenter` (implicit: any segment with `'C'` as an endpoint
  is a center connection)

### Schema migration v12 → v13

```js
function migrateToV13(parsed) {
  parsed.hexes = parsed.hexes || {};
  Object.values(parsed.hexes).forEach(h => {
    if (!Array.isArray(h.overlays)) h.overlays = [];
    h.overlays.forEach(o => {
      if (o.type === 'water') {
        o.segments = [];
        delete o.edges;
        return;
      }
      // For river/road/route, build segments defensively from any
      // legacy edges array. Given no real data exists, this is a
      // belt-and-suspenders pass.
      if (!Array.isArray(o.segments)) {
        o.segments = legacyEdgesToSegments(o);
      }
      // Drop legacy fields
      delete o.edges;
      delete o.flowFromEdges;
      delete o.trunkPair;
      delete o.originAtCenter;
    });
  });
}

function legacyEdgesToSegments(overlay) {
  const edges = Array.isArray(overlay.edges) ? overlay.edges : [];
  const inflows = new Set(overlay.flowFromEdges || []);
  const wantsCenterOrigin = !!overlay.originAtCenter;
  if (edges.length === 0 && !wantsCenterOrigin) return [];
  if (edges.length === 0 && wantsCenterOrigin) {
    // Lone center marker — not representable in segments alone, so
    // emit a degenerate "stub" segment from C to itself? No: model
    // forbids self-loops. Emit empty segments and let the user
    // re-add via editor. Log a warning.
    console.warn('migrateToV13: lone center origin dropped', overlay);
    return [];
  }
  if (edges.length === 1) {
    const e = edges[0];
    const isInflow = inflows.has(e);
    if (overlay.type === 'river') {
      return [{ from: 'C', to: e, flow: isInflow ? 'toFrom' : 'fromTo' }];
    }
    return [{ from: 'C', to: e }];
  }
  if (edges.length === 2 && !wantsCenterOrigin) {
    const [a, b] = edges;
    if (overlay.type === 'river') {
      const aIn = inflows.has(a), bIn = inflows.has(b);
      let flow = null;
      if (aIn && !bIn) flow = 'fromTo';      // a -> b
      else if (bIn && !aIn) flow = 'toFrom'; // b -> a
      // both inflow or both outflow or neither: leave flow null
      return [{ from: a, to: b, flow }];
    }
    return [{ from: a, to: b }];
  }
  // 3+ edges, OR 2 edges with center-origin: fan out from C
  return edges.map(e => {
    const seg = { from: 'C', to: e };
    if (overlay.type === 'river') {
      seg.flow = inflows.has(e) ? 'toFrom' : 'fromTo';
    }
    return seg;
  });
}
```

This migration is exhaustive but mostly dead code given no real
data — it's there to handle any test data that might exist in
localStorage from the exploratory phase, and to document the
mapping for posterity.

### State additions

No new persistent `state.ui` fields needed.

### Naming / CSS conventions

- Module-level cache prefixed `_overlayCache*` (preserved from 14.8).
- New CSS classes prefixed `s-*` for segment editor UI (e.g.
  `.s-segment-row`, `.s-segment-add`).
- Anchor point sentinel: literal string `'C'`. Use as-is in JSON,
  no special encoding.

---

## Stage 1 — Schema v13 + segment graph builder + run finder

Goal: schema migrated, the graph builder produces correct connected
runs from the new segment model, end-to-end testable via devtools
even though no rendering or UI changes are visible yet.

This stage produces NO visible UI changes. Test via devtools console
inspection and direct state mutation.

### What to add

1. **Schema bump v12 → v13**:
   - Change `schemaVersion: 12` → `schemaVersion: 13` everywhere.
   - In `load()`, add `if (v < 13) migrateToV13(parsed);` to chain.
   - Implement `migrateToV13` and `legacyEdgesToSegments` per the
     section above.
   - At the end of `load()`, after migration, verify each overlay
     has the expected v13 shape. If not, log a warning.

2. **Anchor-point geometry helpers**:
   ```js
   // Returns world-space {x, y} for a given anchor of a hex.
   // anchor is 'C' or an integer 0..5.
   function anchorPoint(hex, anchor) {
     if (anchor === 'C') return hexCenter(hex);
     return edgeMidpointWorld(hex, anchor);
   }
   ```
   `hexCenter` and `edgeMidpointWorld` already exist from prior
   slices; reuse them.

3. **Per-type-offset helper for segment endpoints**:
   The slice 14.7 / 14.8.1 offset rule (rivers always one side,
   roads other side, routes further out) applies to edge anchors
   only. Center anchors get NO offset (they're at the hex center;
   no perpendicular reference axis). So:

   ```js
   function anchorPointWithOffset(hex, anchor, overlayType) {
     if (anchor === 'C') return hexCenter(hex);
     // Reuse 14.8.1's globally-consistent offset math
     return offsetEndpoint(hex, anchor, overlayType);
   }
   ```
   `offsetEndpoint` is from slice 14.8.1; reuse.

4. **Segment graph builder**:
   ```js
   // For a given overlay type ("river", "road", "route"),
   // return:
   //   nodes: Map<nodeKey, { hexKey, anchor, x, y }>
   //          where nodeKey = "col,row|anchor"
   //          and "anchor" is 'C' or '0'..'5'
   //   edges: array of { a: nodeKey, b: nodeKey,
   //                     overlayType, hexKey, segment }
   //          where segment is the full segment object {from, to, flow}
   function buildOverlayGraph(type) { ... }
   ```
   - Walk every hex, every overlay of the given type, every segment.
   - For each segment `{from, to}` in hex H:
     - Add node `H|from` if not present.
     - Add node `H|to` if not present.
     - Add an internal graph edge between them, tagged with the
       segment.
   - For each pair of adjacent hexes (A, B) sharing edge `e_a` in A
     / `e_b` in B (where `e_b = (e_a + 3) % 6` for pointy-top):
     - If both `A|e_a` and `B|e_b` exist as nodes (i.e. both hexes
       have a segment touching that shared edge), they are the SAME
       physical point. Add an external graph edge between them with
       no segment tag (it's a hex-boundary continuity edge, not a
       drawn segment).
   - Center anchors have NO external connections — they're internal
     to a single hex.

5. **Run finder**:
   ```js
   function findRuns(graph) {
     // Returns array of Run objects.
     // A Run is a maximal connected path through the graph.
     // Endpoints of a run are nodes with degree 1, OR junction
     // nodes (degree >= 3) — a junction is a run endpoint, and
     // each branch off the junction is a separate run.
     // For each Run, return { nodes: nodeKey[], segments: segment[] }
     //   where segments[i] is the segment associated with the
     //   internal graph edge between nodes[i] and nodes[i+1]
     //   (or null if it's a hex-boundary edge).
   }
   ```
   - Walk the graph from each unvisited degree-1 or degree-≥3 node.
   - Follow degree-2 nodes through the run.
   - Stop when hitting another degree-1 or degree-≥3 node.
   - Mark traversed graph edges as visited; subsequent runs skip them.
   - For purely cyclic networks (every node degree-2), pick any node
     as the start, walk until returning, mark as a closed run.

6. **Devtools-accessible test harness**:
   - Expose `window._wmGraphTest = { buildOverlayGraph, findRuns,
     segmentsForHex }` so I can poke at it from devtools.
   - This is a debug aid; remove or comment out after Stage 1 ships
     if you want, but leaving it in is fine.

### Constraints

- Do NOT touch the existing renderer in this stage. The render code
  may break (because `edges` no longer exists on overlays), and that
  is expected — Stage 2 fixes it. Leaving the renderer broken
  through Stage 1 is acceptable; the user is testing graph-build
  correctness via devtools, not visual output.
- If the existing renderer crashes on load (because it reads
  `overlay.edges` which is now undefined), wrap the renderer's
  overlay-drawing block in a `try/catch` for the duration of Stage 1.
  Print a one-line warning to console. Stage 2 removes the try/catch.

### Test checklist for Stage 1

All tests are devtools-driven.

1. **Migration runs cleanly**:
   - Reload after deploy. State migrates to v13. No errors in console.
   - `localStorage.getItem('wm_unified_v1')` parses to
     `schemaVersion: 13`.

2. **Empty state shape**:
   - Manually inspect `state.hexes` — every overlay has `segments:
     []`, no `edges`/`flowFromEdges`/`trunkPair`/`originAtCenter`.

3. **Manual segment injection via devtools**:
   ```js
   // In devtools:
   state.hexes['3,3'] = state.hexes['3,3'] || { terrain: 'plains', overlays: [] };
   state.hexes['3,3'].overlays.push({
     type: 'river',
     segments: [
       { from: 0, to: 3, flow: 'fromTo' },
     ],
   });
   save();
   ```

4. **Graph build correctness**:
   ```js
   const g = window._wmGraphTest.buildOverlayGraph('river');
   console.log(g.nodes.size, g.edges.length);
   // Expect: 2 nodes, 1 edge for the above
   ```

5. **Adjacent-hex connectivity**:
   ```js
   // Add a continuation in the neighbor across edge 0
   state.hexes['3,2'] = state.hexes['3,2'] || { terrain: 'plains', overlays: [] };
   state.hexes['3,2'].overlays.push({
     type: 'river',
     segments: [{ from: 3, to: 1, flow: 'fromTo' }],
   });
   save();
   const g = window._wmGraphTest.buildOverlayGraph('river');
   // Expect: 4 nodes, but with an external edge connecting
   // "3,3|0" and "3,2|3" — verify in g.edges that both nodes are
   // reachable from each other.
   ```

6. **Run finder**:
   ```js
   const runs = window._wmGraphTest.findRuns(g);
   // For the 2-hex straight river: expect 1 run with 4 nodes
   // (3,2|1 -> 3,2|3 -> 3,3|0 -> 3,3|3).
   ```

7. **Junction**:
   ```js
   state.hexes['3,3'].overlays[0].segments = [
     { from: 'C', to: 0, flow: 'toFrom' },   // inflow at 0
     { from: 'C', to: 2, flow: 'fromTo' },   // outflow at 2
     { from: 'C', to: 4, flow: 'fromTo' },   // outflow at 4
   ];
   save();
   const g2 = window._wmGraphTest.buildOverlayGraph('river');
   const runs2 = window._wmGraphTest.findRuns(g2);
   // Expect 3 runs, each with 2 nodes (one ending at C, the other
   // at an edge or continuing into adjacent hex).
   ```

8. **Parallel non-connecting segments in same hex**:
   ```js
   state.hexes['4,4'] = state.hexes['4,4'] || { terrain: 'plains', overlays: [] };
   state.hexes['4,4'].overlays.push({
     type: 'road',
     surface: 'dirt',
     segments: [
       { from: 4, to: 5 },   // road A: W to NW corner-ish
       { from: 0, to: 2 },   // road B: NE to SE
     ],
   });
   save();
   const g3 = window._wmGraphTest.buildOverlayGraph('road');
   // Expect 4 nodes, 2 edges — TWO disconnected runs.
   const runs3 = window._wmGraphTest.findRuns(g3);
   // Expect runs3.length === 2.
   ```
   This is the headline new capability — confirm it works.

9. **Existing non-overlay features still work**:
   - All non-map tabs (factions, npcs, events, etc.) function.
   - Map tab loads (terrain, hex selection); rendering of overlays
     may be missing/broken, that's acceptable for Stage 1.

---

## Stage 2 — Renderer rewrite, smooth turns, lake-blocks-roads, z-order fix

Goal: replace the existing overlay renderer entirely with a
segment-aware version. Rendering should be visibly correct: smooth
turns, parallel non-connecting paths, no road-deletion when adding
ponds/lakes, lakes block road segments that cross center.

This stage will be large — likely 300+ lines. **Pause partway and
check in if it grows beyond expected scope.**

### What to add / change

1. **Remove old overlay renderer**:
   - Find the slice 14.8 overlay-rendering block.
   - Delete the graph builder, run finder, spline drawer, endpoint
     circle drawer, flow arrow drawer — all of them.
   - Keep the per-type offset constants and the `offsetEndpoint`
     helper (from 14.7 / 14.8.1) — those still apply to edge
     anchors.

2. **New renderer**, calling Stage 1's `buildOverlayGraph` and
   `findRuns`:

   - For each overlay type in z-order (rivers, ponds, roads,
     lakes-outline, routes — see z-order section below):
     - Build the graph.
     - Find runs.
     - For each run, build a list of waypoints by walking nodes:
       - For each node `H|anchor`, compute waypoint via
         `anchorPointWithOffset(H, anchor, overlayType)`.
     - Render the run as a centripetal Catmull-Rom spline through
       the waypoints. Use the same spline implementation from 14.8.
   - Render endpoint circles at each run's terminal nodes (degree-1
     nodes), suppressed where the terminal is a center inside a
     hex with a water overlay or where the terminal is at a
     water-terrain hex's edge.
   - Render flow arrows at each river run's terminal nodes (rules
     below).

3. **Smooth turns are automatic**. A 2-segment pass-through stored
   as `[{from: 1, to: 4}]` in a single hex produces a run with
   waypoints at (edge 1 offset point, edge 4 offset point) plus
   adjacent hex neighbors at the boundaries. There's NO hex center
   waypoint, so the spline curves smoothly without dipping toward
   the center. This is the headline visual win of the slice.

4. **Junctions work via center anchors**. A junction stored as
   `[{from: 'C', to: 0}, {from: 'C', to: 2}, {from: 'C', to: 4}]`
   produces three runs, each with a center waypoint. Splines meet
   at the center. No `trunkPair` logic needed — the segment
   structure encodes the trunk implicitly (in this case, all three
   are equal-weight branches). For the more common case of a
   3-edge junction with one trunk:
   - User would store as `[{from: 1, to: 4}, {from: 'C', to: 2}]`
     to express "pass-through 1↔4 with branch coming in at 2" —
     where the branch connects to the trunk at C? No, that doesn't
     work — the branch's `'C'` and the trunk's path don't share a
     node.

   Actually, this is a real model question. Let me resolve it now:

   **Junction model rule**: To express "a Y-junction where the
   trunk passes through 1↔4 and a branch joins at 2", the segments
   must share a node. The user stores it as:
   ```
   [{from: 1, to: 'C'},
    {from: 'C', to: 4},
    {from: 'C', to: 2}]
   ```
   Three segments all sharing `'C'`. The runs are:
   - Run A: 1 → C → 4 (the trunk, 2 graph edges in 1 run)
   - Run B: C → 2 (the branch, 1 graph edge)
   - Wait, that's not quite right either — `'C'` has degree 3 in
     this hex's subgraph, so it's a junction node. Run finding
     stops at degree-≥3 nodes. So we get THREE runs: 1→C, C→4,
     C→2. The trunk 1↔4 is split into two runs at the junction.
     That's correct behavior — each "branch off the junction"
     including the trunk's two halves is its own run.
   - For smoothness: the renderer should detect that two runs
     meeting at a junction are a "logical trunk" if they have
     mutually-opposite incoming/outgoing tangents at the junction
     (i.e. they continue each other). Render them as one continuous
     spline crossing the junction; render the third run (the
     branch) as a separate spline that peels off with tangent-
     matching at the center (similar to slice 14.8 stage 1's branch
     handling). Use the same `0.3 * trunkTangent + 0.7 * branchNormal`
     formula from 14.8.

   The detection rule: at a junction node, compute the incoming
   tangent from each connected run. If two runs have opposite
   tangents (within some angular tolerance, say 30°), merge them
   visually as a continuous spline. Other runs at the junction
   render as branches with tangent-matching to the merged trunk.

   This gives clean Y-shapes without needing manual trunk overrides.

5. **Endpoint circles**:
   - At each degree-1 node of a run, render a 5px filled circle.
   - Color: same as overlay type, ~20% darker.
   - Suppress where:
     - The node is a center anchor in a hex with a water overlay.
     - The node is an edge anchor and the hex has water terrain.
     - The node is an edge anchor at the outer rim of the map (no
       neighbor exists). Render the circle in this case (river
       running off-map).

6. **Flow arrows (rivers only)**:
   - For each river run's terminal nodes, render a small arrow
     based on the segment's `flow` field at that end of the run.
     - If the terminal node is the `from` end of a segment with
       `flow === 'fromTo'`: arrow points OUT (away from center).
     - If the terminal node is the `to` end of a segment with
       `flow === 'fromTo'`: arrow points IN (toward center).
     - If `flow === 'toFrom'`: reverse the above.
     - If `flow === null`: no arrow at this terminal.
   - Arrow direction comes from the spline tangent at the terminal,
     not the edge normal (preserves the slice 14.8 stage 3 behavior).
   - Arrow size: 6px. Color: river color, ~30% darker.
   - Suppress arrows on internal junction nodes (degree ≥ 3) — only
     terminal nodes get arrows.
   - Suppress arrows where endpoint circles are suppressed (water).

7. **Z-order fix**:
   ```
   1. terrain fill
   2. rivers (splines + endpoint circles + flow arrows)
   3. ponds (rendered as a circle at hex center)
   4. roads (splines + endpoint circles)
   5. lakes (rendered as a wobbly blob at hex center, OUTLINE on top of
      roads to make lakes "block" roads visually)
   6. routes (red dashed splines)
   7. path-highlight (gold preview from travel calc)
   ```

   Critical change vs. previous z-orders: **lakes render LAST among
   non-route overlays**, on top of roads. This makes lakes visually
   block roads. Ponds render BENEATH roads so roads cross over them
   like bridges.

8. **Lake-blocks-roads behavior**:
   - The data model permits a hex to have both a road overlay and a
     lake overlay — that's fine, no destructive deletes.
   - The renderer enforces visual blocking: any road segment that
     has `'C'` as an endpoint (i.e. the road passes through or ends
     at the hex center) is **suppressed during rendering** if the
     hex also has a `lake` overlay. Pure edge-to-edge road segments
     (not through center) render normally — the road "goes around"
     the lake.
   - The lake's blob covers the suppressed segment visually anyway,
     so even if the rule misfires the worst case is "you see a
     lake."
   - The travel calculator (slice 14.5) is updated to mirror this:
     - A hex with a `lake` overlay AND a road segment that has
       `'C'` as endpoint → road bonus does NOT apply; lake terrain
       rules apply (impassable).
     - A hex with a `lake` overlay AND ONLY edge-to-edge road
       segments (no center) → road bonus applies (road goes around).
     - A hex with a `pond` overlay (any size) AND any road →
       road bonus applies (road bridges the pond).

9. **River-into-water clipping**:
   - Preserved from slice 14.8: river spline endpoints inside a hex
     with a water overlay are clipped to the water's bounding circle.
   - With the new model, this affects any river segment whose
     endpoint is `'C'` in a water-overlay hex (the spline's center
     waypoint is replaced with a point on the bounding circle).
   - For multiple rivers entering a lake (multiple river segments in
     the same hex with `'C'` as endpoint), each gets clipped
     independently. The lake blob covers all clipped endpoints,
     producing the "rivers vanish into the lake" visual without
     gaps.

10. **Per-type offset preserved**:
    - Rivers, roads, routes still use the per-type perpendicular
      offset on edge anchors. Center anchors are unoffset (they're
      ALL at the hex center, so offsetting doesn't help and would
      visually misalign).
    - When a river segment connects edge 0 → C, the river's spline
      starts at the offset edge-0 point (rivers' offset) and ends at
      the geometric hex center.
    - When two segments share a center node (e.g. one segment ends
      at C, another starts at C), they meet at the geometric center
      — no offset needed there.

### Constraints

- The renderer must be cache-aware. Reuse the
  `_invalidateOverlayCache()` infrastructure from 14.8. Cache key
  must include segment data; if the cache key was based on `edges`,
  update it.
- Performance target: a map with 50+ overlay hexes renders cache-hit
  in <50ms, cache-miss in <500ms.
- Keep the segment data shape clean. Don't introduce render-only
  fields on segments.

### Test checklist for Stage 2

1. **Smooth turns**: paint (manually via devtools) a 2-segment
   pass-through `{from: 1, to: 4}` in a single hex with neighbors
   continuing on both ends. The spline curves smoothly through the
   hex WITHOUT dipping to the center. Visible win vs. slice 14.8.

2. **Parallel non-connecting roads**: set up two segments in one
   hex `[{from: 4, to: 5}, {from: 0, to: 2}]`. Both render as
   separate splines. Neither connects to the other. They don't
   overlap visually.

3. **Junction Y-shape**: set up three segments
   `[{from: 1, to: 'C'}, {from: 'C', to: 4}, {from: 'C', to: 2}]`.
   The 1↔4 trunk renders as a continuous spline through the center.
   The branch at 2 peels off cleanly with tangent-matching.

4. **Add pond to hex with road** (devtools-set both): pond renders
   beneath the road. Road still visible. No data deletion.

5. **Add lake to hex with edge-to-edge road** (no center): lake
   renders. Road still visible (it goes around the lake — but
   visually the lake blob may cover the road's edge anchors? Test
   what this looks like and report. If the lake blob covers
   significant road, we may need to adjust lake size or the road's
   edge offset).

6. **Add lake to hex with through-center road**: lake renders, road
   segment with `'C'` is suppressed visually.

7. **Travel calc correctness**:
   - Path through a pond+road hex: road bonus applies.
   - Path through a lake+through-center-road hex: lake terrain
     applies (impassable).
   - Path through a lake+edge-to-edge-road hex: road bonus applies.

8. **River into lake (single)**: a hex with a lake and one river
   segment ending at C. River spline clips into the lake bounding
   circle; lake blob covers the clipped end. Smooth visual.

9. **River into lake (multiple)**: same hex with three river
   segments ending at C from different edges. All three clip into
   the bounding circle; all three are covered by the lake blob.
   No visible gaps.

10. **Endpoint circles**: a 1-segment river `{from: 'C', to: 3,
    flow: 'fromTo'}` renders with no endpoint circle at C (it's
    a degree-1 center node — wait, do we want a circle here? It's
    a spring source. Yes, render the circle, slightly larger / 7px
    to distinguish "spring at center" from "edge endpoint." Same
    rule as slice 14.9 was going to add. Bring it in here.) and a
    standard 5px circle at edge 3. Flow arrow at edge 3 pointing
    OUT.

11. **Performance**: stress with 50+ overlay hexes, verify smooth
    rendering and cache behavior.

12. **All previous features preserved**: travel calc, network
    smoothing, water overlays, undo (oh wait — undo isn't shipped
    yet, deferred to 14.11). Confirm slices 14.5, 14.7, 14.8,
    14.8.1 features all still work.

---

## Stage 3 — Painter and eraser rewrite

Goal: paint and erase tools build/modify segments instead of edges.
Continuity-snap, single-edge erase (Shift modifier), drag-trace all
work with the segment model.

This stage will likely be large. **Pause partway and check in if
needed.**

### What to add / change

1. **Remove old painter logic**: the slice 14 stage 1/2 click and
   drag-trace logic that mutated `overlay.edges`. Replace entirely.

2. **New click-paint**:
   - User clicks inside hex H in paintRoad / paintRiver / paintRoute
     mode.
   - Compute the nearest anchor point to the click — could be any
     of the 7 anchors (6 edges + center).
   - Decide what to do:
     - If H has no overlay of this type yet: create a new overlay
       with one segment from this anchor to... what? A click only
       picks one anchor. We need two for a segment.
     - **Click-paint creates a stub from C to the nearest edge.**
       This is the single-click default behavior. If the user wants
       a different segment, they use drag.
     - If H already has an overlay of this type: add a stub from C
       to the nearest edge IF that segment isn't already present.
       Otherwise no-op (or remove if we wanted toggle behavior, but
       erasing is its own mode).

3. **New drag-paint**:
   - On mousedown in hex H: record the nearest anchor as `startAnchor`.
   - On mousemove crossing into a new hex H', or to a different
     anchor in the same hex: add a segment.
     - Same hex, different anchor: add segment `{from:
       prevAnchor, to: newAnchor}` to H's overlay. Update
       `prevAnchor`.
     - Different hex: add segment `{from: prevAnchor, to: e_a}` to
       H (where `e_a` is the edge of H bordering H'), then add
       segment `{from: e_b, to: newAnchor}` to H' (where `e_b` is
       the corresponding edge of H' bordering H). Set `prevAnchor`
       to `newAnchor` for the next move.
   - On mouseup: finalize stroke.

4. **Continuity-snap (preserved from 14.8.1)**:
   - When the painter is about to add a segment endpoint at an
     edge anchor, check: does the hex already have a same-type
     overlay with a segment endpoint at that exact anchor or a
     nearby anchor (within snap distance)?
   - If yes, snap to the existing endpoint instead of creating a
     duplicate.
   - "Nearby" rule: same as 14.8.1 — half the hex apothem in
     world-space pixels.
   - For center anchors: there's only one center per hex; snapping
     is automatic (a click near the center always picks `'C'`).

5. **Flow direction during painting (rivers only)**:
   - The painter records direction of motion.
   - For each new river segment created during a stroke, set
     `flow: 'fromTo'` such that flow follows the direction of
     painting. (e.g. dragging from edge 0 toward edge 3 across a
     hex creates `{from: 0, to: 3, flow: 'fromTo'}`.)
   - For existing segments that the painter snaps to (continuity-
     snap): do NOT override flow. Leave the existing flow intact.
   - For roads/routes: ignore flow entirely.

6. **Single-edge erase (Shift modifier, preserved from 14.8.1)**:
   - With Shift held, click in erase mode: identify the segment
     whose visible spline path is closest to the click point. Remove
     just that segment from the overlay's segments array.
   - Without Shift: remove the entire overlay of that type from
     the hex (existing behavior).
   - Drag-erase with Shift: remove segments matching the entry/exit
     paths through each hex (mirror of drag-paint).
   - Drag-erase without Shift: remove the whole overlay from each
     hex (existing).

7. **Painter prevents duplicates**:
   - Before adding a segment, check: is `{from: A, to: B}` or
     `{from: B, to: A}` already in this overlay's segments? If
     yes, no-op.

8. **Painter prevents self-loops**:
   - Don't add `{from: X, to: X}`. (Shouldn't happen, but
     defensive.)

9. **Modes preserved**:
   - paintRoad, paintRiver, paintRoute — all use the same
     drag/click logic, just different overlay types.
   - eraseRoad, eraseRiver, eraseRoute — Shift modifier toggles
     full vs. single-segment.

10. **Map mode toolbar updated**:
    - No new buttons. The existing toolbar buttons drive the new
      logic transparently.
    - Tooltip updates: paint buttons say "Click for stub, drag to
      trace network." Erase buttons say "Click for full overlay,
      Shift+click for single segment."

### Constraints

- The painter and erasers must NOT touch overlays of other types.
  Painting a road never modifies a river overlay in the same hex.
- Cache invalidation must fire on every paint/erase action.
- All new segments default `flow: null` for non-rivers; for rivers,
  `flow` is set per the painting direction rule.

### Test checklist for Stage 3

1. **Click-paint creates stub**: click "Paint Roads", click in an
   empty hex near edge 2. Renders a stub from C to edge 2.

2. **Drag-paint pass-through**: drag from outside hex through hex
   entering at edge 1 and exiting at edge 4. Hex gets `{from: 1,
   to: 4}` segment — no center waypoint, smooth turn.

3. **Drag-paint junction**: paint a long road through a hex,
   then paint a branch ending at the same hex. The branch's segment
   should connect to the existing road via continuity-snap or via
   the user explicitly drawing through C. Test both: with snapping,
   the branch's endpoint snaps to one of the existing edge anchors;
   without snapping, drawing into a hex's center creates a
   `{from: edge, to: 'C'}` segment.

4. **Parallel non-connecting roads**: paint a road from edge 4 to
   edge 5 in one hex. Then paint a separate road from edge 0 to
   edge 2 in the same hex. Both segments coexist; renderer shows
   them as parallel non-connecting curves.

5. **Continuity-snap**: paint a stub at edge 0 in hex H. Drag a
   second road into hex H from a direction near edge 5. If edge 5
   is within snap distance of edge 0, the painter snaps to edge 0
   and the result is a single connected road, not a fork.

6. **Flow direction on river paint**: drag-paint a river from edge
   0 toward edge 3. The created segment has `flow: 'fromTo'`. The
   inflow visual shows at edge 0 (arrow pointing in), outflow at
   edge 3 (arrow pointing out).

7. **Single-edge erase (Shift)**: paint a 3-edge road junction
   (3 segments all meeting at C). Shift+click on one of the spokes
   in erase mode. Just that one segment is removed; the other two
   remain.

8. **Full erase**: same junction. Click WITHOUT Shift. Whole
   overlay removed.

9. **No cross-type modification**: paint a river and a road through
   the same hex. Erase the road. River is intact.

10. **Defensive duplicate prevention**: try to paint the same
    segment twice. Second attempt is a no-op.

11. **Refresh persistence**: all painted segments persist correctly
    across reload.

---

## Stage 4 — Detail editor rewrite + flow synchronization button + docs

Goal: detail editor exposes the full segment model. Users can add,
remove, and edit segments per overlay. Manual "synchronize flow"
button propagates flow direction across connected river segments.
Doc updates.

### What to add / change

1. **Remove old detail editor controls** for overlays:
   - The 6 edge checkboxes per overlay.
   - The "Trunk pair" dropdown.
   - The "Flow from" / "Inflow edges" controls.
   - The "Origin at center" checkbox (was planned for 14.9; not
     shipped; this slice subsumes that work).
   Replace entirely with the new segment editor.

2. **New segment editor per overlay subsection**:
   - Section header: "Segments".
   - List of current segments, one row each:
     ```
     [from-anchor] → [to-anchor]   [flow]   ✕
     ```
     - `from-anchor` and `to-anchor` are dropdowns with options
       "Center" and "NE / E / SE / SW / W / NW" (matching the slice
       14 directional naming). Changing either dropdown rewrites
       the segment.
     - `flow` (rivers only): three-state toggle "from→to / from←to /
       —". Changing it sets `flow` to `'fromTo'` / `'toFrom'` /
       `null`.
     - ✕ button removes the segment.
   - "Add segment" button at the bottom: adds a new segment with
     defaults (`from: 'C'`, `to: 0`, flow null) and selects the
     first dropdown for editing.
   - Validation: if the user picks the same anchor for `from` and
     `to`, show a warning and don't save until they fix it.
   - Validation: if the user creates a segment that duplicates an
     existing one, show a warning and don't save.

3. **"Synchronize flow" button (rivers only)**:
   - Below the segment list in the river subsection: a button
     labeled "Sync flow across network".
   - Clicking it walks the connected river network starting from
     this hex's overlay, and propagates flow direction to all
     segments with `flow: null`.
   - Algorithm:
     - Find any segment in the network with non-null flow. Call its
       direction "canonical."
     - BFS from there: for each connected segment, set its flow to
       match the canonical direction (whichever orientation aligns
       water flow consistently).
     - Segments with conflicting non-null flow: leave alone, log a
       console warning naming the conflicting hexes.
   - If no segment has flow set, the button does nothing (or shows
     a tooltip "Set flow on at least one segment first").
   - Confirmation prompt before propagating: "This will set flow
     direction on N connected segments. Continue?"

4. **Suppress UI elements that no longer apply**:
   - Water subsection: no segment editor (water overlays have
     `segments: []` and are configured by `size` only — pond/lake
     radio buttons remain unchanged).

5. **`docs/data-model.md` updates**:
   - Bump every reference to v12 → v13.
   - Replace the Overlay block with the new shapes:
     ```js
     // River
     { type: "river", segments: [{from, to, flow}, ...] }
     // Road
     { type: "road", surface: "dirt"|"gravel"|"stone",
       segments: [{from, to}, ...] }
     // Route
     { type: "route", segments: [{from, to}, ...] }
     // Water
     { type: "water", size: "pond"|"lake", segments: [] }
     ```
   - Document the segment shape:
     ```
     Segment: { from, to, flow? }
     - from, to: 'C' (hex center) or edge index 0..5 (NE/E/SE/SW/W/NW)
     - flow (rivers only): 'fromTo' | 'toFrom' | null
     - Constraints: from !== to. No two segments in one overlay
       have the same {from, to} unordered pair.
     ```
   - Document the rendering rules:
     - Splines pass through anchor offset points (edges) and the
       hex center (for C anchors). 2-segment pass-throughs without
       a C anchor produce smooth turns.
     - Junctions (3+ segments sharing an anchor): one pair is
       auto-detected as the trunk based on tangent opposition;
       others render as branches with tangent-matching.
     - Lake-blocks-roads: a road segment with `'C'` as an endpoint
       is suppressed visually and excluded from travel-calc bonus
       in any hex with a lake overlay.
   - Add the v12 → v13 migration note (trivial since no data).
   - Document the consolidation: "Slice 14.10 replaced the
     edge-based overlay model with a segment-based model. Slices
     14 through 14.9 were the exploratory phase. Subsumed: 14.9
     (center origin, now expressed as `from: 'C'`). Deferred to
     14.11: map undo system."

6. **`ROADMAP.md`**:
   - Mark slice 14.9 as "Dropped — subsumed by 14.10".
   - Mark slice 14.6 as "Rescheduled to 14.11".
   - Move slice 14.10 to Done.
   - Update the header.

### Constraints

- The segment editor must validate before saving. Don't write
  invalid segments (self-loops, duplicates) to state.
- The "Synchronize flow" button is per-overlay (i.e. lives in the
  river subsection of one hex's detail panel) but operates on the
  full connected network across hexes.

### Test checklist for Stage 4

1. **Segment editor add**: open hex detail panel, add a road
   segment via "Add segment" button. Segment appears with default
   `from: 'C'`, `to: 'NE'`. Renderer updates immediately.

2. **Segment editor edit**: change `from` from C to E. Renderer
   updates.

3. **Segment editor remove**: click ✕ on a segment. Removed.
   Renderer updates. If overlay has zero segments, overlay is
   removed.

4. **Self-loop validation**: try to set both dropdowns to same
   anchor. Warning shown, save blocked.

5. **Duplicate validation**: try to create two segments with same
   `{from, to}`. Warning shown, save blocked.

6. **Flow toggle (rivers)**: cycle through fromTo / toFrom / null.
   Flow arrows update.

7. **Sync flow button**: build a connected river network across 5
   hexes. Set flow on one segment manually. Click "Sync flow".
   Confirmation appears. Confirm. All connected segments get
   matching flow direction. Conflicting segments (if any pre-set
   conflicting) are logged.

8. **Doc diff**: v13 throughout, segment shape documented, lake-
   blocks-roads rule documented, consolidation history documented.

9. **ROADMAP diff**: 14.9 dropped, 14.6 rescheduled, 14.10 done.

10. **Regression**: travel calc, water overlays, network smoothing,
    z-order all behave correctly.

### Completion summary

- Total lines added/changed in `index.html`.
- Confirmation slice 14.10 is complete.
- Note any decisions on auto-detection rules (junction trunk
  detection by tangent opposition, etc.) where you tuned the
  thresholds.
- Confirm the segment model is the canonical shape going forward
  and old edge-based fields are fully removed from the schema.
