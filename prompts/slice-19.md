# Slice 19 — Map fixes

## Goal
A grab-bag of map polish from the "Planned Patches / Features" list:
- Hex name font recolour for better contrast on light terrains.
- Yellow halo around roads for visibility.
- Double-click a hex → opens its local map.
- River-crossing travel penalty (+0.25 days unless bridged).
- Lake-passage travel penalty (+0.25 days for transit through a lake hex).

## Non-goals
- Narrative-event recording stays manual for now. Players-aware relation
  cycles, clock fills, and quest terminal transitions still commit silently
  — the GM can still create Events by hand if they want a record.
- No "Deceased" toggle / Graveyard — separate item on the patch list.
- No "Draw / Edit travel route" tool — separate patch.
- No new bridge data structure (see schema note).

## Schema
No changes. Schema stays at v14. **No localStorage backup needed.**

Bridge semantics note:
- Stage 3 detects bridges *implicitly* using existing road and river data —
  if a route crosses a river inside a hex AND a road segment in the same hex
  also crosses that river, the road is treated as a bridge and the penalty
  is suppressed. No `bridge: true` flag on segments, no schema change.
- If the implicit rule turns out to mis-classify cases the GM cares about, a
  later patch could add explicit per-segment bridge data — explicitly out
  of scope here.

## Stages

### Stage 1 — Visual polish
Three small visual edits to the world map.

**Hex name font.** In `drawWorldHex` (or wherever hex names are stencilled
onto the canvas), improve contrast on light terrains. Two cheap options —
pick whichever looks best when you eyeball it:
- Switch the fill color to a darker tone, OR
- Keep the existing color but add a 1px text-shadow / outline pass via
  `ctx.strokeStyle` + `strokeText` before `fillText`.
Test against `plains`, `settled`, `desert`, and the unexplored fog overlay.

**Road yellow halo.** In `_renderNetworkType` for `type === 'road'`, before
stroking the existing road color, stroke the same path with a slightly wider
warm-yellow line (e.g. `#d4b87a` or similar tone from the existing palette,
`lineWidth: typeWidth.road + 2`). The result: roads pop visibly against
forest / dark-forest / mountain tiles without changing colour identity.

**Double-click hex → local map.** Add a `dblclick` handler on the world map
canvas. On double-click, resolve the hex via `pxToHex` and open its local
map via the existing `openLocalMap(hexKey)` function. Suppress the default
single-click-select behaviour for that event so we don't both select and
open. (Single-click select must keep working unchanged.)

Acceptance:
- Hex names are readable on every terrain, including unexplored fog.
- Roads are visibly distinguished from rivers and routes; the yellow halo
  doesn't ride over rivers (z-order is `rivers → ponds → roads` per
  `data-model.md`, so this is automatic).
- Double-click opens the local map; single-click still selects the hex.
- Existing ruler / paint / erase modes are unaffected by the dblclick
  handler.

### Stage 2 — Travel calc: river-crossing penalty
**Rule (per the GM):**
- Penalty is per-hex, not per-edge-pair-between-hexes.
- For each hex on the route path, check inside that hex: does the route's
  path through the hex *cross* a river segment in the hex?
- *Cross* = the route's chord through the hex and the river's chord through
  the hex topologically intersect inside the hex. Travel parallel to the
  river (route and river run alongside each other, no crossing) does NOT
  trigger the penalty.
- *Bridge* = if a road segment in the same hex also crosses the same river
  segment, the road bridges it and the penalty is suppressed for that river.
- Per crossed-and-unbridged river: +0.25 days. Multiple unbridged crossings
  in one hex stack (rare but allowed).

**Implementation sketch.** Walk `pathKeys`. For each hex at index `i`,
derive the route's entry anchor and exit anchor through that hex:
- `i === 0` (start): entry = `'C'`, exit = shared edge with `pathKeys[1]`.
- `i === last`: entry = shared edge with `pathKeys[i-1]`, exit = `'C'`.
- otherwise (transit): entry = shared edge with `pathKeys[i-1]`, exit =
  shared edge with `pathKeys[i+1]`.

Use existing `_sharedEdge(prevHexCoord, curHexCoord)` to find the shared
edge index; remember to flip `(eIdx + 3) % 6` to get the entry side from
the cur hex's perspective vs the exit side from the prev hex's perspective.

Then test "do these two segments cross inside the hex?" using a small
helper:

```js
// Both segments are {from, to} on a hex's anchor space (edges 0-5 or 'C').
// Returns true iff their paths through the hex intersect.
segmentsCrossInHex(routeSeg, riverSeg)
```

Cases:
- Both segments are edge-to-edge (`{a, b}` where `a, b ∈ 0..5`): they cross
  iff their endpoints interleave on the cyclic edge order 0..5. (Standard
  chord-interleaving test on a circle.)
- One segment is `{C, e}` (a center-spoke) and the other is `{a, b}`
  (edge-to-edge): the spoke crosses the chord iff `e` is strictly between
  `a` and `b` on the *shorter* arc of the hex perimeter. (Geometric
  intuition: the chord separates `M(e)` from `C` only when `e` is in the
  small arc.)
- Both segments touch `C`: they share an endpoint. Treat as NOT crossing
  (they meet at center but don't impede each other).

Apply this helper in two places per hex:
1. For each river segment, check `segmentsCrossInHex(routeSeg, riverSeg)`.
   If true → this river is crossed.
2. For each crossed river, scan road segments in the same hex; if any road
   segment also satisfies `segmentsCrossInHex(roadSeg, riverSeg)` → this
   river is bridged in this hex.

Sum unbridged crossings → multiply by 0.25 → add to the hex's days
contribution.

**Edge cases & decisions.**
- Start/end hex (route anchor includes `'C'`): the "route segment" is a
  half-chord from C to the entry/exit edge. Apply the same `segmentsCross`
  helper — a `{C, eOut}` route vs an edge-to-edge river still tests
  cleanly; a `{C, eOut}` route vs a `{C, eR}` river touches at C only =
  not crossing. So endpoints participate naturally; no special-casing.
- A river segment with `from === to` is invalid by schema constraint, so
  no degenerate case there.
- If a single river segment is crossed by both a road AND the route in
  the same hex, that's a bridged crossing → no penalty.

**Ruler panel display.**
- Update `daysForPath` to return both `days` and a per-hex breakdown that
  includes any river-crossing penalty added at that hex.
- The existing per-hex Days column should reflect the penalty: a hex with
  modifier 1.0 and one unbridged crossing shows `1.25 / move + 0.25`.
- Optional polish: add a small ⚠ marker or "(+R)" tag in the per-hex row
  when a crossing penalty applied. Skip if it bloats the stage.

Acceptance:
- A two-hex straight-line path that has a river running across the
  destination hex (segment perpendicular to the route's travel direction)
  adds 0.25 to total days.
- Same scenario but a road segment in that hex also crosses that river the
  same way → no penalty (bridged).
- A path that runs parallel to a river — both heading the same direction
  through the same hex without crossing — adds nothing.
- A start hex with a river crossing the way out: the penalty applies in
  whichever hex the actual crossing happens (could be the start hex if
  the river splits start exit from path).
- Existing "all-water path" warning and modifier-override features still
  work.

### Stage 3 — Travel calc: lake-passage penalty
For each *interior* hex in the path (not first or last), if the hex has a
`water` overlay with `size === 'lake'`, add +0.25 days. Endpoints (start /
destination) don't trigger — you're arriving or departing, not transiting.

Wire into the same `daysForPath` rewrite from stage 2. The per-hex Days
column should reflect the lake penalty for transit lake hexes.

(Note: the existing lake-blocks-roads rule in `modifierFor` already
suppresses the road bonus when a lake forces the road through center —
this is independent and continues to work. The +0.25 here is on top of
whatever modifier `modifierFor` returns.)

Acceptance:
- A path that transits a lake hex (interior) adds 0.25.
- A path that *starts* or *ends* on a lake hex doesn't add the lake
  penalty for that endpoint.
- A path with N transit-lake hexes adds 0.25 × N.
- Lake-blocks-roads logic is unaffected — a road through a lake hex still
  gets road modifier suppressed when its segments touch C.

## Test plan (final manual smoke)
1. Name a few hexes across plains, forest, mountain, desert, unexplored.
   Confirm legibility.
2. Paint roads across forest and mountain hexes. Confirm yellow halo is
   visible and doesn't bleed onto rivers.
3. Double-click a hex → local map opens. Single-click → just selects.
4. Build a hex with a single river segment going edge 0 ↔ edge 3 (across
   the hex). Run a ruler path entering edge 1 and exiting edge 4 (the
   route chord interleaves with the river chord) → confirm +0.25.
5. Same hex, paint a road also going edge 0 ↔ edge 3 (overlapping the
   river path). Re-run the ruler → confirm no penalty (bridged).
6. Paint a road going edge 1 ↔ edge 4 in the same hex (parallel to the
   route, perpendicular to the river — does NOT cross the river). Confirm
   penalty re-applies (the road runs alongside, doesn't bridge).
7. Build a hex with a river spoke `{C, 0}`. Run a ruler path 1 ↔ 5 (a
   short chord in the small arc near edge 0) — confirm crossing detected
   → +0.25.
8. Same spoke `{C, 0}`, run path 2 ↔ 4 (chord in the long arc, away from
   edge 0) → confirm no crossing.
9. A path that starts in a hex with a river and exits via an edge that
   doesn't cross the river → no penalty in start hex.
10. Pick a path that transits a lake hex (interior). Confirm +0.25.
11. Pick a path that *starts* or *ends* on a lake hex (no transit).
    Confirm no lake penalty.
12. A path with both a river crossing and a lake transit → both penalties
    stack additively.

## Out of scope (deferred)
- "Deceased" toggle and Graveyard archive — separate slice/patch.
- Draw / Edit travel route tool — separate patch.
- Explicit per-segment `bridge: true` flag on roads — only if the
  implicit rule proves insufficient.
- Narrative-event recorder of any flavour — explicitly dropped from this
  slice; remains manual.

## Workflow
- Commit `prompts/slice-19.md` first.
- Stage-based: explicit "go" confirmation between stages.
- One commit per stage. Stage 2 is the riskiest (geometry); the
  `segmentsCrossInHex` helper is a good sub-commit boundary if the diff
  gets large. A small dev test harness that exercises the chord-crossing
  cases in the console is worth attaching to the helper, parallel to the
  `_wmGraphTest` precedent from slice 14.10.
- Update `ROADMAP.md` to move slice 19 from Planned → Done at the end.
  The slice line should now read just "Slice 19: Map fixes" (drop the
  "Narrative-event recorder +" prefix). Update the header to "as of
  slice 19 complete". Remove the patch list entries that this slice
  fulfilled (hex font, road halo, double-click, river crossing, lake
  passage).
- Update `data-model.md` only if anything in the schema actually changes
  (it shouldn't). If we ever document the implicit-bridge rule formally,
  the Overlay rendering / lake-blocks-roads section is the right neighbour.
