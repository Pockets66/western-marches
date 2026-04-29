# Slice 20 — Pathfinder + route editor

## Goal
Turn the travel calculator from a straight-line estimate into a real
route planner, and let routes track the roads they follow:

- The road bonus becomes context-aware: a road only helps if the
  route can actually travel along it through the hex. Road on the
  wrong side of a lake → no bonus.
- The ruler computes the shortest path with Dijkstra over the new
  cost function instead of taking a straight cube-line.
- A small "stay on roads" tiebreaker prefers contiguous-road paths
  when no obvious shortcut wins on raw days. Roads are worth more
  than travel speed alone (inns, navigation, security), so this nudge
  bakes that intuition into routing without making it absolute.
- Saved routes graphically follow the road segments inside hexes
  where the bonus applied, instead of cutting straight chord through
  every hex.
- Manual route paint/edit/erase tools so the GM can override the
  pathfinder when narrative demands.
- **Saved routes are selectable on the map** — clicking one shows its
  travel time, lets the GM recompute or delete it, and surfaces its
  endpoints with the same A/B colour scheme used during ruler mode.
- Ruler-mode UX polish: hex A is cyan, hex B is magenta, and clicking
  a third hex while both are set starts a fresh measurement.

## Non-goals
- No schema change. Routes are an existing overlay type; the road
  segment graph is already in place from slice 14.10.
- No "weight by faction control" or other strategic-layer routing —
  the cost function stays terrain + road + crossings + lake.
- No multi-waypoint routes inside a single ruler operation. Multi-leg
  journeys are still built by saving multiple routes additively.
- No player-facing route reveal — discovery toggles are separate work.
- No naming or labeling routes as named "journeys" — selectable, yes;
  identifiable as data entities, no.

## Schema
No changes. Schema stays at v15. **No localStorage backup needed.**

(Slice 19 already shipped the road `state` field at v15 and the
surface × state multipliers in `modifierFor`. Slice 20 only changes
how those modifiers are *consumed*, how routes are *drawn*, and how
the canvas *responds to clicks* — the data shape doesn't move.)

A "selected route" identity is computed at click-time from the
connected-run graph, not stored. UI state holds an opaque key that
points back to one of the run's segments; the run is reconstructed on
demand. This means schema-free selection and no stale references when
segments change.

## Stages

### Stage 1 — Context-aware road bonus
Refactor the cost function from "per-hex modifier" to
"per-hex-in-path modifier". Same data, finer question: given a route
entering this hex at anchor `eIn` and leaving at `eOut`, can it
actually travel along the road, and what's its cost?

**Replace `modifierFor(hexKey)`** with two functions:
- `terrainModifier(hex)` — pure terrain/override modifier, no road
  bonus. Returns 0 for water and impassable.
- `costForHexInPath(hex, eIn, eOut, hexKey)` — context-aware. Returns
  `{ daysCost, usedRoadBonus }`. Internally:
  1. Apply ruler override if set (`state.ui.rulerOverrides[hexKey]`).
     If override is `impassable`, return `{ daysCost: Infinity }`.
  2. If terrain is water with no override → `Infinity`.
  3. Try to claim the road bonus: see "Road bonus eligibility" below.
     If granted, modifier = `surfaceMult × stateMult`; else modifier
     = terrain modifier.
  4. Return `{ daysCost: 1.25 / (move × modifier), usedRoadBonus }`.

The two callers — Dijkstra (stage 2) and the ruler-panel display
(stage 2 / stage 5) — use this same function. Ripple-edit
`daysForPath` to consume it.

**Road bonus eligibility.** The bonus applies in a hex iff *all* of:
- Hex has a road overlay.
- Hex is not lake-blocked: either no lake, or the road has at least
  one edge-to-edge segment that doesn't touch `'C'` (existing rule
  from slice 14.10 — preserve).
- The road's segment graph in this hex has a path from `eIn` to
  `eOut`. Both endpoints must be road anchors. "Path" = breadth-first
  reachable through the road's segments inside this hex (degenerate
  one-segment case `{eIn, eOut}` directly is the common one, but
  star/junction configurations should also work).
- Endpoint hexes (route anchor includes `'C'`): the road bonus
  applies iff the road has a segment from `'C'` to the other anchor
  (or the road's graph reaches between `'C'` and the entry/exit edge
  through intermediate anchors).

Expose a small helper `roadPathInHex(hex, eIn, eOut)` that returns
either the segment chain from `eIn` to `eOut` (for stage 3 to consume)
or `null`. Stage 1 uses it for the eligibility test; stage 3 reuses
it to copy segments.

**Surface × state on undiscovered hexes.** When `hex.explored === false`:
- Surface is treated normally (the GM placed the road, so the surface
  is "known terrain" to the planner).
- State defaults to `'average'` (×1.0) regardless of what's stored on
  the overlay. The planner doesn't lean on hidden condition data;
  estimates stay consistent with what a scout would assume about a
  road they've heard about but haven't traveled.

This gate matters most for the pathfinder's choice between routes
through unexplored regions — a stretch of unexplored stone-good road
shouldn't bias the planner with information the players don't have.

**Ruler-panel display update.** The per-hex Days column reflects the
path-specific cost (which depends on adjacent hexes through eIn /
eOut). Add a small marker when a hex has a road but the route didn't
get the bonus — e.g. a faint "(road)" tag with `title="road bypassed
this hex"` so the GM can see why a road hex contributed cross-country
days.

Acceptance:
- A road that runs east–west across a hex: route entering W, exiting
  E → bonus applies. Route entering N, exiting S → no bonus.
- A road on a lake hex with all `C`-touching segments → no bonus
  (existing lake-blocks-roads).
- A road on a lake hex with one edge-to-edge segment that bypasses
  the center, where the route enters and exits via that segment's
  edges → bonus applies.
- An unexplored stone-good road hex contributes `1.5 × 1.0 = ×1.5`
  (using state default), not `1.5 × 1.25 = ×1.875`.
- The same hex once `explored: true` flips to `×1.875`. Re-running
  the ruler reflects the change.
- `roadPathInHex` returns a segment chain for valid eIn/eOut pairs
  and `null` for invalid ones. Suitable for unit-style console
  testing.
- Existing all-water warning and modifier-override features still
  work.

### Stage 2 — Dijkstra pathfinder + dashed live preview + ruler UX
Replace the straight-line `hexLine` in the ruler with Dijkstra. Build
the graph implicitly: nodes = hexes within map bounds; edges = each
hex's six neighbors that exist; edge weight = `costForHexInPath` of
the source hex with `eIn` carried from the prior step and `eOut` =
shared edge with the destination.

**Edge weight subtlety.** Dijkstra typically computes a one-step cost
to a neighbor, but `costForHexInPath` needs both `eIn` and `eOut` to
decide road-bonus eligibility. Resolve by tracking entry-anchor in
the search state:

- State = `{ hexKey, entryAnchor }`. Frontier explores transitions
  to all six neighbors. The cost of moving from `(hex, eIn)` to
  neighbor `n` via shared edge `eX` is the cost of *exiting* the
  current hex with `eOut = eX`, computed via
  `costForHexInPath(curHex, eIn, eX)`. The neighbor's entry anchor
  is `(eX + 3) % 6`.
- Start hex: initial state `(start, 'C')`. End condition: any state
  whose `hexKey === goal`. The final hex's exit cost (with
  `eOut = 'C'`) is added when popping.

Cost ∞ (water + no override, or `impassable` override) → edge
skipped.

**Tiebreaker for contiguous roads.** Apply a `×0.95` factor to the
edge cost when *both* the source and destination of a transition got
the road bonus. This is small enough to be overridden by any
meaningful shortcut (5% saved on one hex won't beat saving a whole
hex's worth of travel) but big enough to settle ties in favor of
staying on roads. Document the multiplier as a single named constant
(e.g. `ROAD_CONTINUITY_BONUS = 0.95`) so it's tunable.

**Disconnected case.** If Dijkstra exhausts the frontier without
reaching the goal, the ruler shows "No path found" instead of the
days table. Replaces the current "Path crosses N impassable hexes"
warning when the path is fully blocked. Remove that warning since
Dijkstra routes around impassable hexes when a route exists; the
warning becomes vestigial.

**Dashed live preview.** While the ruler has both endpoints set,
render the computed path as a dashed red overlay on the canvas
(reuse the existing route render colour `#a02828`, dash pattern
`[5, 4]` already in use). The preview redraws whenever the overrides
change. The preview is not stored — it's purely visual.

**Button rename.** "Mark route on map" → "Save route on map". The
preview is what the user sees; Save persists it. Append behavior
unchanged: pressing Save adds the path's segments to each hex's
route overlay.

**Ruler UX polish.**

*Hex A cyan / hex B magenta.* Wherever `drawWorldHex` (or its caller)
currently highlights `state.ui.rulerHexA` and `rulerHexB`, switch to:
- A: cyan outline (suggested `#00bcd4`, tunable). Drawn as a thicker
  border on the hex, sitting visibly above the terrain fill but
  below pins/names.
- B: magenta outline (suggested `#e91e63`, tunable). Same treatment.
- Existing select-hex highlight (used by plain `select` mode) stays
  whatever it is now — it's a separate concept.

The same colour scheme is reused by selected routes in stage 5 to
highlight their endpoint hexes — keep the colours as named constants
so both stages reference one source of truth.

*Click-resets-when-both-set.* Replace the current rulerA → rulerB
auto-advance flow with a unified click handler:
- In `rulerA` or `rulerB` mode, clicking a hex:
  1. If `rulerHexA` is null → set A, switch to `rulerB` mode.
  2. Else if `rulerHexB` is null → set B, render panel (stay in
     `rulerB` mode).
  3. Else (both set) → clear A and B, set new A as the clicked hex,
     switch to `rulerB` mode (waiting for next click as B).
- The "click a third hex restarts" behavior gets the GM out of
  having to toggle the ruler off and back on between measurements.

Acceptance:
- A path with a clear road shortcut takes it.
- A path through forest where a small road detour saves time picks
  the road; a path where the detour saves nothing significant ignores
  it.
- Equal-cost paths with and without a road choose the road
  (tiebreaker).
- Disconnected start/end (e.g. an island with no bridge) shows "No
  path found".
- Dashed preview appears as soon as both endpoints are placed and
  updates when overrides change.
- Save still uses the existing append behavior — multiple saves
  build a multi-leg journey.
- Hex A renders cyan-outlined, hex B magenta-outlined, while in
  ruler mode.
- Clicking a third hex while both are set clears B and starts a
  fresh measurement from the new click.

### Stage 3 — Route follows roads graphically
When saving a route, for each hex where the route used the road
bonus, copy the road's segments along the `eIn → eOut` road path
into the route overlay. For hexes where the bonus didn't apply, fall
back to the current single `{eIn, eOut}` segment.

**Implementation sketch.**
- During Dijkstra (stage 2), record per-hex
  `(eIn, eOut, usedRoadBonus)` on the winning path.
- For each hex on the saved path:
  - If `usedRoadBonus`: call `roadPathInHex(hex, eIn, eOut)` (helper
    from stage 1). The returned segment chain is the route's segment
    chain in this hex — copy each road segment as a route segment
    with the same `from`/`to`. Don't share segment objects; routes
    and roads are independent overlays.
  - Else: add a single `{from: eIn, to: eOut}` route segment as
    today.
- The renderer's per-type +6px offset (route at `+6` vs road at `+3`)
  already keeps them visually parallel — no render change required.
  The dashed route line tracks the road's curve through junctions
  and turns instead of cutting a chord.

**Edge case — endpoint hexes.** Start and end hexes have one anchor
as `'C'`. The road graph may need to traverse `{C, eOut}` or
similar. Same algorithm; `'C'` is a valid anchor in road segments.

**Edge case — multiple road segments between eIn and eOut.** If the
graph finds more than one path (a road junction inside the hex gives
two ways to get from edge 0 to edge 3), pick the shortest path in
segment count. Stable tie-break: prefer paths through `'C'` if any.

Acceptance:
- A road runs through a hex via segments `{0, C}, {C, 3}` (Y-junction
  spoke). A route entering edge 0 and exiting edge 3 saves with
  segments `{0, C}, {C, 3}` — the route renders bending toward the
  junction, parallel to the road.
- The same hex with a road `{0, 3}` (straight pass-through) saves
  the route as `{0, 3}` — parallel straight line.
- A hex where the road bonus didn't apply (e.g. road on the wrong
  side of a lake) saves a single straight `{eIn, eOut}` route
  segment as before.
- Multiple "Save route on map" presses still build up additional
  legs without overwriting prior routes.

### Stage 4 — Manual route paint / edit / erase
Mirror the road and river painting tools so the GM can override or
augment routes by hand.

**Toolbar buttons.** Add `paintRoute` and `eraseRoute` modes
alongside the existing `paintRoad`/`paintRiver`/`eraseRoad`/
`eraseRiver` buttons. Same drag-paint behavior as roads (drag from
one anchor to another; crossing into a neighbor hex automatically
adds an exit segment in the source hex and entry segment in the
destination). Erase shift+drag removes individual segments touching
the dragged edge; plain drag removes whole route overlays per-hex
(matches road erase behaviour).

**Segment editor entry.** Add a `route` row to the segment editor in
the hex info panel, parallel to the existing road and river rows.
Same add-segment / remove-segment / anchor pickers as roads. Routes
have no `surface`, no `flow`, no `state` — just `segments`.

**Cascade rules.** Routes are leaves; no entity references them. No
new cascade work needed. They participate in `_pushMapUndo` /
`_invalidatePinCache` like any other overlay edit.

**Re-rendering and offsets.** The `+6px` perpendicular offset for
routes already keeps them visually distinct from roads and rivers on
shared edges (slice 14.8 / 14.10). No render changes.

Acceptance:
- Paint a route from hex A across to hex B; renders dashed red.
- Erase a route segment via shift+drag; rest of the route survives.
- Plain-drag erase across multiple hexes clears the route in each
  hex.
- Segment editor lets a hex's route segments be edited individually.
- Manually-painted routes persist across reloads and don't interfere
  with the saved-from-pathfinder routes (both are just route
  segments).

### Stage 5 — Route selection + info panel
Saved routes can be clicked to select them. Selecting a route shows
its travel time and gives the GM tools to recompute or delete it.

**Selection state.** New UI state:
- `state.ui.activeRouteSegmentRef: { hexKey, fromAnchor, toAnchor } | null`
  — points at one segment of the selected run. Stable enough to
  re-resolve the connected run from, even if other route segments
  elsewhere change. If the referenced segment is deleted (e.g. by
  segment editor) `activeRouteSegmentRef` is cleared.
- The connected run is recomputed on demand from this seed segment
  by walking the route graph (BFS over segment endpoints, including
  cross-hex edge-anchor continuations — same graph the renderer
  builds via `buildOverlayGraph('route')`).

**Hit-test in select mode.** When the user clicks the canvas in
`select` mode:
1. Resolve click position to world coordinates.
2. For each route segment in any hex, compute the segment's two
   anchor world positions (with the route +6 offset). Test distance
   from click to the line segment between them. If within ~8px,
   it's a hit.
3. If multiple segments hit, pick the one whose distance is
   smallest.
4. On hit: clear `_mapSelHex`, set `activeRouteSegmentRef` to that
   segment's reference, render the route info panel, redraw to
   apply selection highlight.
5. On miss: existing hex-select behavior. Also clear
   `activeRouteSegmentRef` so selecting a hex deselects any active
   route.

Spline-curvature hit-testing (sampling the rendered Catmull-Rom
points) is more precise but more work. The straight-line hit-test
between anchor world positions is good enough — segments are short
relative to a hex, and the spline rarely deviates more than a few
pixels from the chord at the +8px tolerance scale.

**Selection highlight.** While a route is selected, render its
connected run with a wider gold halo behind the existing dashed red
stroke. Reuse the gold tone used for selected pins
(approximately `#c8972a` — match exactly to the existing constant).
Endpoint hexes (degree-1 nodes of the run) get the same cyan/magenta
hex outlines from stage 2: cyan for whichever endpoint is "first"
(stable by hexKey lex order — opaque to the user, just needs to be
deterministic), magenta for the other.

**Route info panel.** Reuse the existing ruler panel as a shared
"travel panel" with two modes:
- *Ruler mode:* `rulerHexA` and `rulerHexB` set, no
  `activeRouteSegmentRef` — current Stage-2 behavior. Save button.
- *Selected mode:* `activeRouteSegmentRef` set —
  - Title: "Selected route" (vs current "Travel Time").
  - Reconstruct the route's hex sequence by walking the connected
    run from one endpoint to the other via the route segment graph.
    For each transit hex, derive eIn/eOut from the run; for endpoint
    hexes, anchor is `'C'` paired with the appropriate edge.
  - Compute days using `costForHexInPath` per hex — same as ruler.
  - Show the same per-hex breakdown table.
  - Buttons: "Recompute path" (re-runs Dijkstra between the run's
    endpoints, deletes all current segments of this run, saves the
    new path's segments — useful when the GM updated terrain since
    the original save), "Delete route" (cascades through the run's
    segments and clears them all in one undo unit), "Clear selection"
    (deselects without deleting anything).
- Both modes share the same per-hex Days table layout and the same
  `(road)` marker convention.

**Per-hex modifier override interaction.** Per-hex modifier overrides
(`state.ui.rulerOverrides`) only apply during *active ruler mode*;
the selected-route panel ignores them and uses ground-truth
modifiers. Document that. (If the GM wants to test "what if this hex
were impassable", they should re-measure with the ruler, not select
an existing route.)

**Mutual deselection (sanity).**
- Clicking a route → `_mapSelHex = null` and
  `activeRouteSegmentRef = {…}`.
- Clicking a hex (in select mode, no route hit) →
  `activeRouteSegmentRef = null`, normal hex select.
- Entering ruler mode → `activeRouteSegmentRef = null`,
  `_mapSelHex = null`. Conversely, clicking a route while in ruler
  mode does nothing (route hit-testing is gated to `select` mode
  only) — preserves the click-resets behavior of stage 2.

Acceptance:
- Click on a saved route in select mode → route highlights gold,
  panel opens with travel time, endpoints get cyan/magenta hex
  outlines.
- Selected hex (if any) deselects when route is selected.
- Clicking another route → previous route deselects, new route
  selects.
- Clicking empty space (no route, no hex) → both deselect.
- "Recompute path" replaces the route's segments with a fresh
  Dijkstra path between its current endpoints. Map state changes
  (new road, terrain edit) reflect in the recomputed path.
- "Delete route" removes all segments of the connected run in a
  single undo step.
- "Clear selection" deselects without altering the route.
- A route's travel time is correctly computed on selection, even if
  the route was painted manually (stage 4) — graph walk handles both
  origins identically.
- A route created from a single hex (one segment) is selectable;
  endpoints are the segment's `from` and `to`. Time is just that
  hex's contribution.
- Selecting a route while in ruler mode is impossible (gated). A
  route selected from before ruler mode entered persists in panel
  state? — No. Entering ruler mode clears the selection. Document
  this.

## Test plan (final manual smoke)
1. Build a forest map with a stone-good road running east–west
   through row 0. Place ruler endpoints in the same row east and
   west of the road's stretch. Confirm path takes the road and Days
   reflects ×1.5.
2. Place ruler endpoints north and south through that road's hex
   without using its east-west axis. Confirm "(road)" marker
   appears and the per-hex modifier reverts to forest ×0.5.
3. Place a lake adjacent to a road, force the road's segments
   through `'C'` (so lake blocks). Run a route through that hex.
   Confirm no road bonus, "(road)" marker, terrain modifier applies.
4. Same lake hex, but the road has an edge-to-edge segment skirting
   the lake. Route entering and exiting via those edges → bonus
   applies. Route entering elsewhere → no bonus.
5. Mark a hex as `explored: false` with a stone-good road. Re-run
   the ruler. Confirm cost matches stone-average (×1.5), not
   stone-good (×1.875). Mark the hex `explored: true`; re-run;
   confirm ×1.875.
6. Two equal-distance paths to the same destination — one across
   plains all the way, one half-on-road half-cross-country with
   identical effective speed. Confirm the road path wins on the
   tiebreaker.
7. Disconnect start from end with a wall of water (no land bridge).
   Confirm "No path found" message; no preview drawn.
8. Place ruler endpoints; confirm dashed-red live preview appears.
   Change a per-hex override; confirm preview updates.
9. Hex A renders cyan, hex B renders magenta during ruler mode.
10. With both endpoints set, click a third hex. Confirm B clears,
    new A is the third hex, ruler advances to rulerB mode.
11. Save the route. Confirm the persisted route follows the road's
    curvature through a junction hex (Y-shaped road), not a
    straight chord.
12. Save a second route on top — confirm append behaviour, both
    routes coexist on the canvas.
13. Switch to select mode. Click on the saved route. Confirm:
    - Route highlights gold.
    - Panel shows travel time with per-hex breakdown.
    - Endpoints display with cyan and magenta hex outlines.
    - If a hex was selected before, it deselects.
14. Edit a hex's terrain that the selected route passes through.
    Click "Recompute path" in the panel. Confirm route updates;
    panel travel time refreshes.
15. Click "Delete route" on a selected route. Confirm all its
    segments vanish in one operation; undo restores the entire
    route.
16. Use the new paintRoute tool to draw a manual route through a
    few hexes. Switch to select mode and click it. Confirm it's
    selectable; travel time computes correctly.
17. Shift+drag-erase a single segment of the selected route. If
    the erased segment was the seed reference,
    `activeRouteSegmentRef` should clear and the panel close. If
    the erase split the route into two disconnected runs, the
    selection follows the run that still contains the seed
    segment (or clears if the seed was on the erased segment).
18. Open the segment editor for a hex with a route; add and remove
    segments via the picker. Confirm changes persist and render.
    If the selected route's segments are touched, selection state
    updates appropriately (clear if seed gone, keep if seed
    survives).

## Out of scope (deferred)
- Player-facing route reveal toggle (filtering routes by discovery).
- Multi-waypoint routes inside a single ruler operation.
- Faction-control or political weighting on route cost.
- Saving routes as named/labeled "journeys" rather than raw
  overlays.
- Auto-suggesting routes between settlements without user input.
- Explicit per-segment `bridge: true` flag — only if needed.
- Spline-precise hit-testing (vs straight-line anchor hit-testing).
  Bump the tolerance if the simpler test misses obvious cases.

## Workflow
- Commit `prompts/slice-20.md` first.
- Stage-based: explicit "go" confirmation between stages.
- One commit per stage. Stage 2 (Dijkstra + UX) is the largest and
  most invasive — consider sub-commits for "search algorithm" /
  "preview + button rename" / "cyan-magenta + click-resets" if the
  diff balloons. Stage 3 depends on state from stage 2's search
  bookkeeping; ship them in order. Stage 5 depends on stage 1's
  `costForHexInPath` and stage 2's panel structure.
- No localStorage backup needed (no schema change).
- Update `ROADMAP.md` to move slice 20 from Planned → Done.
  Suggested line: "Slice 20: Pathfinder + route editor — Dijkstra
  over context-aware road bonus, road-continuity tiebreaker, dashed
  live preview, route-follows-road rendering, manual paint/erase/
  segment-editor for routes, route selection with travel-time
  readout + recompute/delete, ruler hex A/B in cyan/magenta, click
  resets". Update header to "as of slice 20 complete". Remove the
  patch list entries this slice fulfilled (Draw/Edit travel route,
  Route calculator finds shortest path, Mark route → Save route +
  dashed line).
- Update `data-model.md`:
  - The "modifierFor" reference in the lake-blocks-roads section
    should be updated to point at `costForHexInPath`. The lake rule
    itself doesn't change.
  - Document the road-continuity tiebreaker constant near the
    surface×state multiplier table.
  - Document the unexplored-hex state-defaulting rule (state =
    `'average'` when `hex.explored === false`, regardless of stored
    value) somewhere visible — probably under the Road overlay
    schema.
  - Note that route selection is a UI-state-only feature; routes
    themselves remain raw overlay segments.
