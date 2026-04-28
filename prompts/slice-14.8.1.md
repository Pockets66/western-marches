# Slice 14.8.1: Overlay editing patches

## How to work this file

This slice has three independent patches in a single stage. Implement
all three, test each individually, then report when finished.

When done, STOP and report:
- A summary of each patch.
- What I should test for each.
- "Slice 14.8.1 complete." as the final line.

Make surgical edits only. Each patch is small.

## Slice context

After slice 14.8 shipped, three correctness/UX issues surfaced:

1. **Edge offsets sometimes flip when networks merge.** Per-hex
   tangent computation produces inconsistent offset points across
   hex boundaries — adjacent hexes referencing the same physical
   edge can end up offset on opposite sides. With network smoothing
   the spline tries to interpolate through both points and produces
   visible kinks or jogs.
2. **Drag-painting through a hex with existing endpoints creates
   unwanted forks.** The painter computes entry/exit edges
   geometrically, ignoring whether the hex already has same-type
   endpoints nearby. If you drag into a hex where a river already
   ends at edge 2 and your geometric entry is edge 1, the result
   is a Y-junction in that hex even though you wanted to extend
   the existing river.
3. **Erasing requires removing the entire overlay from a hex.**
   To clean up an accidental fork, you have to wipe a hex
   completely and repaint. No way to remove just one edge.

This is a **pure UI/rendering slice**. No schema change. Schema stays
at v12. No backup needed.

## Patch 1 — Globally consistent edge-offset math

### Problem

Slice 14.7 introduced per-type perpendicular offsets so rivers,
roads, and routes don't render on top of each other on shared edges.
The implementation computes the offset along the edge tangent, where
tangent = perpendicular CCW of the (hex center → edge midpoint)
vector.

This per-hex computation is direction-dependent. For two adjacent
hexes A and B sharing a physical edge:

- A's edge `n` has center-to-edge vector pointing one way.
- B's edge `(n+3) % 6` has center-to-edge vector pointing the
  opposite way (180° flipped, since they're on opposite sides of
  the same physical edge).
- "CCW perpendicular" of opposite vectors gives opposite tangents.
- `offset * tangent` along opposite tangents lands on opposite sides
  of the edge.

Result: hex A places its offset point 3px above the edge midpoint,
hex B places its offset point 3px below. Same physical edge, two
different offset points, ~6px apart. Spline tries to fit both →
kink.

### Fix

Compute the offset using a property of the edge that does NOT depend
on which hex is asking. Use the edge's two world-space corner
vertices.

Implementation:

1. **Add a helper `edgeCorners(hex, edgeIdx)`** if one doesn't
   already exist. Returns `[corner0, corner1]`, the two world-space
   vertices of the edge, in any consistent order. (Order can depend
   on the hex; we'll normalize next.)

2. **Add a helper `edgeOffsetVector(corner0, corner1)`** that
   returns a unit vector perpendicular to the edge, pointing in a
   GLOBALLY CONSISTENT direction:
   ```js
   function edgeOffsetVector(c0, c1) {
     const dx = c1.x - c0.x;
     const dy = c1.y - c0.y;
     // Perpendicular: rotate 90° CCW
     let nx = -dy;
     let ny =  dx;
     // Normalize to unit length
     const len = Math.hypot(nx, ny);
     nx /= len; ny /= len;
     // Globally consistent sign: ensure the offset vector always
     // points the same way regardless of corner ordering.
     // Rule: if nx < 0, flip both. If nx === 0, use ny < 0 to flip.
     // This guarantees the same edge always produces the same
     // offset vector, regardless of which hex computed it.
     if (nx < 0 || (nx === 0 && ny < 0)) {
       nx = -nx; ny = -ny;
     }
     return { x: nx, y: ny };
   }
   ```
   The "if nx < 0 flip" rule is arbitrary but consistent. It makes
   `edgeOffsetVector` a pure function of the edge's geometry, not
   of which hex called it.

3. **Replace `offsetEndpoint`** (or whatever the current per-spoke
   offset function is named) to use the new approach:
   ```js
   function offsetEndpoint(hex, edgeIdx, overlayType) {
     const [c0, c1] = edgeCorners(hex, edgeIdx);
     const m = { x: (c0.x + c1.x) / 2, y: (c0.y + c1.y) / 2 };
     const v = edgeOffsetVector(c0, c1);
     const offset = OVERLAY_OFFSET[overlayType];
     return {
       x: m.x + v.x * offset,
       y: m.y + v.y * offset,
     };
   }
   ```

4. **Verify** every call site that computes per-edge positions for
   overlay rendering uses the new helper. The endpoint circles, the
   spline waypoints, the flow arrow positions — all should use
   `offsetEndpoint`. Search the file for hardcoded edge-midpoint
   computations in overlay rendering and refactor through the helper.

### Test for Patch 1

- Paint a road that turns 90° in the middle (e.g. enters hex A's
  edge 1, exits hex A's edge 4). The road renders smooth at the
  hex boundary on both ends.
- Paint a river that converges with another river at a junction.
  At the junction, both river splines reach the same offset point
  on each shared edge. No visible kink.
- Paint a road and a river through the same hex, sharing an edge.
  They render on opposite sides of the edge midpoint, NOT on top
  of each other, AND the offset is consistent — no drift between
  hexes.
- Compare to slice 14.8 baseline: kinks at boundaries should be
  gone.
- Stress-test: paint a complex curving river network across 10+
  hexes. No segment should appear offset from its neighbors.

## Patch 2 — Continuity-snap in painter

### Problem

The drag-trace painter adds the geometrically-computed entry/exit
edges as the cursor crosses hex boundaries. It doesn't consider
existing overlay state in the destination hex. This means dragging
toward an existing river endpoint creates a Y-junction instead of
extending the existing river.

### Fix

When the painter is about to add an edge to a hex's overlay, check
first: does the hex already have an overlay of this type with an
endpoint within "snap distance" of the target edge? If yes, use the
existing endpoint instead of the computed one.

"Endpoint" here means: an edge in the overlay that has no
continuation in an adjacent hex. (A 1-edge stub is an endpoint. A
2-edge pass-through is not — both edges already continue.)

"Snap distance" = half the hex apothem in world-space pixels.

Implementation:

1. **Add a helper `findSnappableEndpoint(hex, type, targetEdgeIdx)`**:
   ```js
   function findSnappableEndpoint(hex, type, targetEdgeIdx) {
     const overlay = hex.overlays?.find(o => o.type === type);
     if (!overlay) return null;
     const targetMidpoint = edgeMidpointWorld(hex, targetEdgeIdx);
     const apothem = HEX_APOTHEM_PX;
     const snapDist = apothem * 0.5;
     let bestEdge = null;
     let bestDist = Infinity;
     for (const edgeIdx of overlay.edges) {
       if (edgeIdx === targetEdgeIdx) continue;        // already there
       if (hasContinuation(hex, edgeIdx, type)) continue; // not an endpoint
       const m = edgeMidpointWorld(hex, edgeIdx);
       const d = Math.hypot(m.x - targetMidpoint.x, m.y - targetMidpoint.y);
       if (d < snapDist && d < bestDist) {
         bestDist = d;
         bestEdge = edgeIdx;
       }
     }
     return bestEdge;
   }
   ```
   `hasContinuation(hex, edgeIdx, type)` returns true if the
   neighboring hex across that edge has an overlay of the same type
   with the corresponding edge present (i.e. this edge is part of
   the network, not a terminal). Compute via the same adjacency math
   slice 14 stage 2 used: edge `n` of hex maps to edge `(n+3) % 6`
   of neighbor. The `hasContinuation` helper either already exists
   or should be added now.

2. **Hook into the paint logic.** Find where the drag-trace
   computes the entry edge for `currHex` (the hex being entered).
   Wrap the geometric edge computation:
   ```js
   const geometricEdge = computeEntryEdge(lastHex, currHex);
   const snappedEdge = findSnappableEndpoint(currHex, type, geometricEdge);
   const finalEdge = snappedEdge !== null ? snappedEdge : geometricEdge;
   ```
   Same for the exit edge from `lastHex` (when you cross into a new
   hex, the painter adds an exit edge to `lastHex` too — apply
   snapping there as well).

3. **Click-paint (single click without drag)** should also benefit
   from snapping. When the user clicks inside a hex in paint mode,
   the painter computes the nearest edge and adds it. Apply the same
   snap check: if the nearest geometric edge is within snap distance
   of an existing endpoint, use the existing endpoint instead.

### Constraints

- Snapping only applies when ADDING edges. Erasing is unaffected.
- Snap to endpoints only — don't snap to internal pass-through
  edges (those already have continuations and snapping would change
  the network topology in unintended ways).
- The snap distance threshold (half apothem) is a tunable. Don't
  add it to `state.ui` — it's a code constant.

### Test for Patch 2

- Paint a 1-edge river stub ending at hex A's edge 2. Then drag
  paint from hex B (across the map) approaching hex A from a
  direction that would geometrically enter hex A through edge 5.
  But edge 5 is close to edge 2 (within half apothem). The painter
  snaps to edge 2 — the rivers connect into a single continuous
  network instead of forking.
- Paint two separate rivers that each end at adjacent edges in the
  same hex. Drag a third river toward that hex from a direction
  that's near both endpoints. The closer one wins (deterministic).
- Paint a river that ends at hex A's edge 0. Drag from hex B
  approaching hex A through edge 3. Edge 3 is across the hex from
  edge 0 (not within snap distance). No snap; geometric entry edge
  3 is used as expected.
- Drag through hex with existing 2-edge pass-through (no endpoints).
  No snapping happens — pass-through edges aren't endpoints. Painter
  may add new edges if the geometric path differs.

## Patch 3 — Single-edge eraser (Shift modifier)

### Problem

Erase modes today wipe the entire overlay of the targeted type from
each affected hex. To remove an accidental fork, you have to erase
the whole hex's overlay and repaint everything else.

### Fix

Holding Shift while clicking or dragging in any erase mode switches
to "single-edge erase": only the nearest edge to the click/cursor is
removed from the overlay, not the whole overlay.

If after removal the overlay has zero edges, remove the overlay
entirely (consistent with current behavior elsewhere — empty
overlays don't persist).

Implementation:

1. **Track the Shift key state**. Two ways:
   - Read `e.shiftKey` directly off the mousedown / mousemove
     events. Cleanest.
   - Track in a module-level boolean via keydown/keyup. More
     code, no benefit.
   Use `e.shiftKey`.

2. **Modify the erase handler**:
   ```js
   function handleEraseClick(hex, type, e) {
     if (e.shiftKey) {
       eraseSingleEdge(hex, type, nearestEdgeTo(e));
     } else {
       eraseFullOverlay(hex, type);   // existing behavior
     }
   }
   ```
   `nearestEdgeTo(e)` computes the nearest edge to the cursor's
   world-space position (already exists or trivially derivable from
   existing edge-distance code).

3. **`eraseSingleEdge(hex, type, edgeIdx)`**:
   ```js
   function eraseSingleEdge(hex, type, edgeIdx) {
     const overlay = hex.overlays?.find(o => o.type === type);
     if (!overlay) return;
     overlay.edges = overlay.edges.filter(e => e !== edgeIdx);
     // Defensive trim: remove flowFromEdges entries pointing at
     // the removed edge.
     if (overlay.flowFromEdges) {
       overlay.flowFromEdges = overlay.flowFromEdges.filter(
         e => e !== edgeIdx
       );
     }
     // Defensive trim: clear trunkPair if it referenced the removed edge.
     if (overlay.trunkPair && overlay.trunkPair.includes(edgeIdx)) {
       overlay.trunkPair = null;
     }
     // Remove overlay if empty.
     if (overlay.edges.length === 0) {
       hex.overlays = hex.overlays.filter(o => o !== overlay);
     }
   }
   ```

4. **Drag-erase with Shift**: same logic but for each hex passed
   through, remove only the entry/exit edge (the same edges that
   drag-paint would ADD). For the start hex, remove just the
   nearest edge to mousedown. For interior hexes, remove the
   entry+exit edges. For the end hex (mouseup), remove the entry
   edge.

   This is the inverse of drag-paint, scoped to single edges.

5. **Wire up undo (slice 14.6)**: single-edge erase still produces
   ONE undo entry per stroke. The existing `beginStroke` /
   `captureHex` / `endStroke` lifecycle handles this — just make
   sure the erase handlers are still wrapped (they should already
   be from slice 14.6 stage 1).

6. **Cursor / visual feedback**:
   - When Shift is held in any erase mode, change the cursor to
     something distinct (e.g. `cell` or `pointer`) so the user
     knows the modifier is active.
   - Implement via a `mousemove` listener that toggles a CSS class
     on the canvas based on `e.shiftKey`.

7. **Tooltip update**: The "Erase Roads" / "Erase Rivers" / "Erase
   Routes" toolbar buttons should mention the Shift modifier.
   Tooltip example: "Erase roads (Shift+click for single edge)".

### Constraints

- Shift modifier only affects ERASE modes, not paint modes. Shift
  in paint mode does nothing extra.
- Single-edge erase respects the same overlay cleanup rules as the
  detail editor: empty overlays are removed, dangling
  `flowFromEdges` and `trunkPair` are trimmed defensively.
- Undo restores the FULL pre-stroke state, including the case where
  a stroke removed multiple single edges across multiple hexes.

### Test for Patch 3

- Paint a 3-edge road junction in one hex. Switch to "Erase Roads".
  Click on one of the spokes WITHOUT Shift → entire road overlay
  removed (existing behavior).
- Repaint the junction. Click on one spoke WITH Shift → only that
  one edge removed; the other two remain as a 2-edge pass-through.
- With a 2-edge pass-through, Shift+click on one edge → 1-edge stub
  remains. Shift+click on the last edge → overlay removed entirely.
- Drag-erase WITHOUT Shift through 5 road hexes → all 5 hexes
  cleared.
- Drag-erase WITH Shift through 5 road hexes → entry+exit edges
  removed from each hex; some hexes may keep partial overlays if
  they had additional edges not touched by the drag path.
- Undo a single-edge erase stroke → all removed edges restored.
- Verify a 3-edge river junction with `flowFromEdges: [0, 2]`:
  Shift+click on edge 0 → edge 0 removed AND
  `flowFromEdges` becomes `[2]`. No render errors.
- Verify Shift+click on a trunk edge with manual `trunkPair` set:
  `trunkPair` becomes null (defensive trim).

## Final report

After all three patches:

- Total lines added/changed in `index.html`.
- Confirmation of each patch's test checklist.
- Any ambiguity encountered (e.g. helper functions that didn't
  exist where expected — note where you added them).
- "Slice 14.8.1 complete."
