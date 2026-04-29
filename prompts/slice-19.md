# Slice 19 — Map fixes

## Goal
Map polish from the "Planned Patches / Features" list, plus a new road
condition system and a render-bug fix uncovered while planning.

## Status going in
**Stage 1 (visual polish) already shipped:**
- Hex names render with a `rgba(0,0,0,0.75)` text outline (`strokeText` →
  `fillText`, 3px round-joined) — readable on every terrain including fog.
- Roads get a per-surface halo behind the main stroke: dirt = `#f5e8c8`
  (cream), gravel = `#c8c8c8` (grey), stone = `#111111` (near-black). The
  halo communicates surface type AND lifts road visibility against
  forest/mountain tiles.
- World-map canvas has a `dblclick` handler that opens the local map for
  the clicked hex; gated to skip overlay-paint and ruler modes so it
  doesn't fight other tools.

**Remaining work (this prompt):** stages 1–4 below.

## Non-goals
- Narrative-event recording stays manual.
- No "Deceased" toggle / Graveyard.
- No "Draw / Edit travel route" tool.
- No pathfinder rewrite — the ruler still uses the existing straight
  cube-line. (Pathfinder + route editor become slice 20.)

## Schema
Bumps to **v15** for the road condition `state` field (stage 2). Stage 1
is pure rendering with no schema implications. **Take a localStorage
backup before stage 2 starts.**

v14 → v15:
- Migration `migrateToV15()`: for every road overlay in `state.hexes`, if
  `state` is missing, set `state = 'average'`. Surface field is already
  in place from slice 14, no work there.
- Increment schemaVersion to 15.

Bridge semantics for stage 3 stay implicit (no `bridge: true` flag); the
crossing test detects bridges from existing road and river segment data.

## Stages

### Stage 1 — Per-hex road surface rendering (bug fix)
Right now `_renderNetworkType` picks `_roadSurf` (and therefore both the
halo colour and the main stroke colour) from a single waypoint —
`wps.find(wp => wp.anchor !== 'C')` — and applies it to the entire
connected run. Result: a road that's stone through a village hex and dirt
out into the wilds renders as one colour across the whole run, depending
on which hex's edge-anchor happened to come first.

Fix it so each portion of a run's spline strokes with the surface colour
of the hex that portion lies in, while keeping the spline geometry global
(the smoothing across hex boundaries from slice 14.10 must be preserved
— don't degenerate to per-hex independent splines).

**Implementation sketch.**
- Compute the spline points exactly as today (centripetal Catmull-Rom
  over the deduped waypoints). Don't change the geometry.
- Walk the spline points pairwise. For each line segment between two
  spline samples, attribute it to a hex. The cleanest attribution is
  the hex of the nearer waypoint (or the waypoint at the segment's
  midpoint — whichever is simpler given the existing
  `hexArrowData` precedent further down in this same function uses
  "destination hex if crossing, shared hex otherwise").
- Stroke each line segment with that hex's road surface colour for
  both the halo pass and the main pass. Halo first, then main, same
  as today.
- The water-terrain center-node split (current `subPaths` logic) still
  applies — split into sub-paths first, then walk segments inside each
  sub-path.

Keep the existing fallback: if a road overlay has no `surface` field,
treat as `'dirt'`.

Acceptance:
- A two-hex road where hex A has surface `'stone'` and hex B has surface
  `'dirt'` renders with the stone tones inside hex A and the dirt tones
  inside hex B — both halo and main stroke. The transition happens at
  the shared edge (give or take one spline sample).
- Curvature across the boundary is unchanged from before — the spline is
  still globally smoothed.
- A road that's all one surface looks identical to before.

### Stage 2 — Road condition `state` (schema v15)
Add a per-overlay-per-hex condition multiplier on top of the surface
multiplier. Surface = "what the road is made of"; state = "what shape
it's in".

**Schema (v14 → v15).**
- Road overlay gains `state: "horrible" | "poor" | "average" | "good"`.
- Default for new and migrated overlays: `'average'`.
- Add migration `migrateToV15()` per the Schema section above.
- Update `data-model.md` Road overlay schema in the same commit.

**Multipliers.**
- Surface: dirt ×1.0, gravel ×1.0, stone ×1.5. (Gravel matches dirt for
  travel purposes — its benefit is durability, which the GM tracks
  manually outside this system.)
- State: horrible ×0.5, poor ×0.75, average ×1.0, good ×1.25.
- Combined multiplier when on a road = surface × state. So:
  stone-good = ×1.875, dirt-average = ×1.0, stone-horrible = ×0.75
  (yes, potholed stone is actually worse than fresh dirt — clean
  multiplication, not min/max).

**`modifierFor()` rewrite.** When the hex has a road overlay AND the
lake-blocks-roads rule does NOT suppress it (existing logic preserved),
return `surfaceMult × stateMult` instead of the current flat `1.0`.
Default missing fields safely (`surface || 'dirt'`,  `state || 'average'`).

**Hex detail UI.** In the road overlay section (where the surface radios
already live), add a parallel set of state radios — same layout, same
naming convention. Writer is `setOverlayState(c, r, state)` mirroring
`setOverlaySurface`.

Acceptance:
- Existing data loads cleanly with every road defaulting to `'average'`.
- Setting a hex's road state to `'good'` or `'horrible'` immediately
  reflects in the ruler-panel days computation for any path through
  that hex.
- Setting it to `'good'` only affects that hex — no propagation.
  (Stage 1 already separated rendering, so this comes for free; the
  schema was already per-hex.)
- Lake-blocks-roads still trumps: a road forced through a lake's center
  reverts to terrain modifier; surface × state is irrelevant in that
  case.
- The hex detail UI shows the current state for the selected hex's road
  overlay; absent road = no radios.

### Stage 3 — Travel calc: river-crossing penalty
Per-hex check inside `daysForPath`. For each hex on the path, does the
route's path through that hex *cross* a river segment in that hex? If
yes and not bridged, +0.25 days per crossed-and-unbridged river segment.

**Crossing rule.**
- *Cross* = the route's chord through the hex topologically intersects
  the river's chord through the hex inside the hex.
- *Parallel* = route and river run alongside without crossing →
  no penalty.
- *Bridge* = a road segment in the same hex also crosses that river
  segment → no penalty for that river.

**Implementation sketch.**
- Walk `pathKeys`. For each hex at index `i`, derive the route's entry
  and exit anchors:
  - `i === 0` (start): entry = `'C'`, exit = `_sharedEdge` to next.
  - `i === last`: entry = `_sharedEdge` from prev, exit = `'C'`.
  - transit: entry from prev, exit to next. Remember to flip
    `(eIdx + 3) % 6` so each hex sees its own side.
- Build `routeSeg = { from: entry, to: exit }`.
- For each river segment in this hex, test
  `segmentsCrossInHex(routeSeg, riverSeg)`. If true → river is crossed.
- For each crossed river, scan road segments in the same hex; if any
  road segment also satisfies `segmentsCrossInHex(roadSeg, riverSeg)`
  → river is bridged.
- Sum unbridged crossings; multiply by 0.25; add to total days.

**Helper: `segmentsCrossInHex(a, b)`.** Both segments are
`{from, to}` over the hex's anchor space (edges 0..5 or `'C'`).
- Both edge-to-edge (`{e1, e2}` and `{e3, e4}` with all four ∈ 0..5):
  cross iff their endpoints interleave on the cyclic edge order
  0,1,2,3,4,5. Standard chord-interleaving test.
- One is a center-spoke `{C, e}`, the other is edge-to-edge `{a, b}`:
  the spoke crosses the chord iff `e` is strictly between `a` and `b`
  on the *shorter* arc of the hex perimeter. Geometric intuition: the
  chord separates `M(e)` from `C` only when `e` is in the small arc;
  if `e` is in the large arc (same side as `C`), the spoke ends on
  the same side as its origin and never reaches the chord.
- Both touch `C` (both segments are spokes): they meet at center but
  don't impede each other; treat as NOT crossing.
- Opposite edges (`{0, 3}`, `{1, 4}`, `{2, 5}`): the chord passes
  through `C`, geometrically degenerate. Pick a tie-break and document
  it. Suggestion: treat as crossing (the road would go through C and
  so would the river).

A small dev-test harness exposing `segmentsCrossInHex` on `window`
(parallel to `_wmGraphTest` from slice 14.10) is worth attaching for
console-driven verification of the geometry cases.

**Ruler panel display.** `daysForPath` should return per-hex breakdown
including any crossing penalty added at that hex. The existing per-hex
Days column should reflect the penalty (modifier days + 0.25 per
unbridged crossing). Optional polish: a small ⚠ marker or "(+R)" tag
in the row when a crossing was counted. Skip if it bloats the stage.

**Edge cases.**
- A river segment with `from === to` is invalid by schema, no degenerate
  case.
- A start/end hex where the route's `'C'` anchor is involved: the
  helper handles it cleanly — `{C, eOut}` route vs edge-to-edge river
  works via the spoke-vs-chord case.

Acceptance:
- Two-hex straight path, destination hex has river `{0, 3}` (across
  perpendicular to the route), route enters edge 1 exits edge 4 → +0.25.
- Same hex with road `{0, 3}` (overlapping the river) → no penalty
  (bridged).
- Same hex with road `{1, 4}` (parallel to route, perpendicular to
  river — alongside the route, doesn't bridge) → penalty re-applies.
- Hex with river spoke `{C, 0}`, route `{1, 5}` (short arc near edge 0)
  → +0.25.
- Same spoke, route `{2, 4}` (long arc, opposite side from edge 0) →
  no penalty.
- Existing all-water warning and modifier-override still work.

### Stage 4 — Travel calc: lake-passage penalty
For each *interior* hex in the path (not first or last), if the hex has
a `water` overlay with `size === 'lake'`, add +0.25 days. Endpoints
don't trigger — you're arriving or departing, not transiting.

Wire into the same `daysForPath` rewrite from stage 3. Per-hex Days
column reflects the lake penalty for transit lake hexes.

(Lake-blocks-roads logic in `modifierFor` is independent and continues
to work — the +0.25 here stacks on top of whatever modifier
`modifierFor` returns.)

Acceptance:
- Path transits a lake hex (interior) → +0.25.
- Path *starts* or *ends* on a lake hex (no transit) → no lake penalty.
- Path with N transit-lake hexes → 0.25 × N.
- Lake-blocks-roads behaviour unchanged.

## Test plan (final manual smoke)
1. Paint a single connected road that runs through three hexes. Set hex
   1 surface to `stone`, hex 2 to `dirt`, hex 3 to `dirt`. Confirm the
   stone tones (black halo + grey body) appear only inside hex 1 and
   the dirt tones (cream halo + brown body) inside hexes 2 and 3.
   Curvature across the boundaries should remain smooth.
2. Set the same road's state to `'good'` in hex 1 and `'horrible'` in
   hex 3. No visual change (state is travel-only). Run a ruler path
   along it; confirm hex 1 contributes more speed and hex 3 contributes
   less than hex 2.
3. Reload after stage 2 ships; confirm existing roads have
   `state: 'average'` and travel times match pre-slice values for any
   all-`'average'` route.
4. Stone road, average state through forest hex: modifier ×1.5.
   Compare against forest-no-road modifier ×0.5 — should be 3× faster.
5. Stone road, horrible state: ×0.75. Confirm the per-hex Days reflects
   this.
6. Reproduce the river-crossing geometry tests (cases 4–8 from the
   prior draft of this prompt; the geometry is unchanged).
7. Reproduce the lake-passage tests (start-on-lake vs transit-lake).
8. Compound case: a path with a stone-good road, a river crossing, and
   a lake transit — verify all three effects stack correctly.

## Out of scope (deferred)
- Pathfinder rewrite (Dijkstra over the new cost function) — slice 20.
- Route draw/edit tools (paintRoute / eraseRoute toolbar buttons + route
  segment editor) — slice 20 or 21.
- Explicit per-segment `bridge: true` flag — only if the implicit rule
  proves insufficient.
- "Mark route on map" → "Save route on map" rename and dashed live
  preview — slice 20.

## Workflow
- Commit `prompts/slice-19.md` first.
- Stage-based: explicit "go" confirmation between stages.
- One commit per stage. Stage 1 is the riskiest visual change (rendering
  rewrite); stage 3 is the riskiest logic change (geometry). Both are
  good candidates for an extra checkpoint commit if the diff balloons.
- **Take a localStorage backup before stage 2 ships** (schema bump).
- Update `ROADMAP.md` to move slice 19 from Planned → Done. The slice
  line should now read just "Slice 19: Map fixes — per-hex road
  rendering, road condition (schema v15), river crossing + lake
  passage travel penalties, hex font + road halo + double-click polish
  (stage 1 shipped pre-slice)". Update header to "as of slice 19
  complete". Remove the patch list entries this slice fulfilled (hex
  font, road halo, double-click, river crossing, lake passage). Leave
  Pathfinder / route draw items in the patch list pointing at slice 20.
- Update `data-model.md`:
  - Road overlay schema gains `state` field.
  - Add v14 → v15 migration entry.
  - Note the surface×state multiplier rule and how it interacts with
    the lake-blocks-roads rule.
