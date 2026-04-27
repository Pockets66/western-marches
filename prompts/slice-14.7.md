# Slice 14.7: Smooth overlay rendering + offsets + bridge calc

## How to work this file

This slice has one stage. When finished, STOP and report:

- A summary of what you changed.
- What I should test before proceeding.
- "Slice 14.7 complete." as the final line.

Do NOT rewrite `index.html` wholesale — make surgical edits only.

## Slice context

The current overlay renderer draws straight lines from each hex
center to each connected edge midpoint. This produces sharp star
patterns at junctions and is visually unsatisfying for a fantasy
map. This slice replaces the straight-line draws with quadratic
Bezier curves and adds per-type perpendicular offsets so rivers,
roads, and routes don't render on top of each other when they share
edges.

Also: the travel calculator currently treats water hexes as
impassable. A hex with BOTH a road AND a river overlay represents a
bridge (road crosses river); this slice updates `modifierFor()` to
allow such hexes to be passable.

This is a **pure rendering / calculation slice**. No schema change,
no migration, no backup required. Schema stays at v11.

### In scope

- Replace straight-line spoke rendering with quadratic Bezier curves
  for all three overlay types.
- Per-type perpendicular offsets (rivers, roads, routes each rendered
  at slightly different perpendicular positions relative to the
  edge midpoint) so they don't overlap visually.
- Cross-hex continuity: an overlay's curve in hex A meets the curve
  in adjacent hex B at the same offset point on the shared edge,
  with matching tangent angles, so the line appears continuous.
- Travel calc bridge semantics: a hex with both `type === "road"`
  and `type === "river"` overlays is treated as passable (road's
  ×1.0 modifier applies).

### Out of scope

- Per-edge or per-segment curve adjustment (no editing of curve shape
  by hand).
- Continuous-network smoothing across many hexes (Technique C from
  earlier discussion). Smoothing is per-hex; cross-hex continuity is
  guaranteed by offset+tangent matching, not by network analysis.
- Rendering an explicit "bridge" graphic (no special bridge sprite,
  no break in the river under the road).
- Schema changes.
- Editing UX changes.

## Implementation details

### Geometry primitives

Two helpers needed in addition to the existing edge-midpoint helper.
Add these alongside the existing hex geometry math.

**`edgeNormal(hex, edgeIdx)`** — returns the unit vector pointing
from the hex center toward the edge midpoint:

```js
function edgeNormal(hex, edgeIdx) {
  const c = hexCenter(hex);                 // existing helper
  const m = edgeMidpoint(hex, edgeIdx);     // existing helper
  const dx = m.x - c.x;
  const dy = m.y - c.y;
  const len = Math.hypot(dx, dy);
  return { x: dx / len, y: dy / len };
}
```

**`edgeTangent(hex, edgeIdx)`** — returns a unit vector perpendicular
to the edge normal (i.e. parallel to the edge itself):

```js
function edgeTangent(hex, edgeIdx) {
  const n = edgeNormal(hex, edgeIdx);
  return { x: -n.y, y: n.x };   // 90° counter-clockwise rotation
}
```

The tangent is used for the perpendicular offset.

### Per-type offset table

```js
const OVERLAY_OFFSET = {
  river:  -3,    // 3px "below" the edge (in tangent direction)
  road:   +3,    // 3px "above" the edge
  route:  +6,    // 6px above — sits beside roads, not under them
};
```

The sign means: which side of the geometric edge midpoint to render.
Offset is applied along the **edge tangent**, NOT in screen-space x/y.

The "above"/"below" framing is just naming; the actual side is
arbitrary as long as it's consistent. What matters is:

1. Rivers and roads are always on opposite sides.
2. Both adjacent hexes apply the same per-type offset rule, so when
   a road crosses from hex A to hex B, both hexes' road curves end
   at the same offset point on the shared edge.

### Offset endpoint helper

For each spoke (one center→edge connection in one overlay), compute
the offset endpoint:

```js
function offsetEndpoint(hex, edgeIdx, overlayType) {
  const m = edgeMidpoint(hex, edgeIdx);
  const t = edgeTangent(hex, edgeIdx);
  const offset = OVERLAY_OFFSET[overlayType];
  return {
    x: m.x + t.x * offset,
    y: m.y + t.y * offset,
  };
}
```

This is the point the curve actually ends at, not the geometric edge
midpoint.

### Curve shape — single spoke

Replace the existing straight-line draw of one spoke (`drawLine(cx,
cy, mx, my, color, width)`) with a quadratic Bezier curve from the
hex center to the offset endpoint, with a single control point that
makes the curve bow gently.

The control point is positioned at 60% of the way from center to
endpoint, offset perpendicular to the spoke direction by a small
amount. The perpendicular offset gives the curve its bow.

```js
function drawSpokeBezier(hex, edgeIdx, overlayType, color, width, ctx) {
  const c   = hexCenter(hex);
  const end = offsetEndpoint(hex, edgeIdx, overlayType);
  // Direction along the spoke
  const dx = end.x - c.x;
  const dy = end.y - c.y;
  // Perpendicular (90° CCW) of the spoke direction
  const px = -dy;
  const py =  dx;
  // Bow magnitude: rivers bow more (natural meander), roads bow less
  const bow = (overlayType === 'river') ? 0.18 : 0.08;
  // Deterministic "wobble" per spoke so adjacent hexes still differ
  const seed = hashString(hexKey(hex) + ':' + edgeIdx + ':' + overlayType);
  const wobble = ((seed % 1000) / 1000 - 0.5) * 2;   // -1..+1
  // Control point at 60% of spoke length, offset perpendicular by bow * wobble
  const ctrl = {
    x: c.x + dx * 0.6 + px * bow * wobble,
    y: c.y + dy * 0.6 + py * bow * wobble,
  };
  ctx.strokeStyle = color;
  ctx.lineWidth = width;
  ctx.beginPath();
  ctx.moveTo(c.x, c.y);
  ctx.quadraticCurveTo(ctrl.x, ctrl.y, end.x, end.y);
  ctx.stroke();
}
```

Adjust the function signature to whatever drawing primitive the
existing renderer uses (canvas 2D context, SVG `<path>` element,
etc.). The math is the same regardless.

**`hashString(str)`** is a small deterministic hash — if the file
already has one, reuse it. Otherwise add a simple one:

```js
function hashString(s) {
  let h = 0;
  for (let i = 0; i < s.length; i++) {
    h = ((h << 5) - h + s.charCodeAt(i)) | 0;
  }
  return Math.abs(h);
}
```

This makes `wobble` deterministic per (hex, edge, overlayType), so
the same hex always renders identically across redraws.

### Important constraints on curve shape

1. **Bow magnitude is small.** 0.08–0.18 of the spoke length. Not
   dramatic. The goal is "soft, natural" not "wiggly."
2. **Wobble is bounded.** The `(seed%1000)/1000 - 0.5) * 2` gives
   roughly `-1..+1`, multiplied by `bow`. So actual bow values land
   between roughly `-bow` and `+bow`. The curve never crosses the
   straight-line path by more than `bow * spokeLength` perpendicular
   pixels.
3. **All spokes within one hex are drawn independently.** No
   smoothing across spokes within a single hex. A 3-edge river is
   three independent curves meeting at the center.
4. **Cross-hex continuity is automatic.** Since both adjacent hexes
   compute the same offset point on their shared edge midpoint
   (using the same `OVERLAY_OFFSET` rule and same edge math), the
   curves meet at that point. Tangent matching at the meeting point
   isn't perfect (each hex's curve is independently bowed), but with
   small bow values the discontinuity is imperceptible.

### Render order (z-order)

Update the renderer to draw overlays in this order, bottom to top:

```
1. terrain fill
2. roads      <- DOWN-most overlay (so rivers render OVER roads)
3. rivers     <- middle overlay
4. routes     <- TOP overlay (treasure-map dashed line, always on top)
5. path-highlight (gold semi-transparent fill from travel calc)
6. pin layer (when slice 15 ships)
```

Wait — check this. The user wants roads ABOVE rivers (bridge effect).
Use this order instead:

```
1. terrain fill
2. rivers     <- bottom overlay
3. roads      <- on top of rivers (bridges)
4. routes     <- always on top
5. path-highlight
6. pin layer (later)
```

This is the order to use. Roads on top of rivers visually implies
a bridge wherever they cross. The bridge calc semantics in
`modifierFor()` (below) make this consistent with travel logic.

### Travel calc bridge semantics

Update `modifierFor(hexKey)` in slice 14.5's calc:

```js
function modifierFor(hexKey) {
  const h = state.hexes[hexKey];
  if (!h) return 0.5;
  const overlays = h.overlays || [];
  const hasRoad  = overlays.some(o => o.type === 'road');
  const hasRiver = overlays.some(o => o.type === 'river');
  // Road bonus — applies even on water terrain when there's also a river
  // (interpreted as a bridge: road crosses river)
  if (hasRoad) return 1.0;
  // Otherwise terrain rules apply
  switch (h.terrain) {
    case 'plains': case 'settled':                             return 1.0;
    case 'hills':  case 'forest':                              return 0.5;
    case 'mountain': case 'swamp': case 'desert':
    case 'dark-forest': case 'ruins':                          return 0.25;
    case 'water':                                              return 0;
    case 'unknown': default:                                   return 0.5;
  }
}
```

Note: previously, `hasRoad` returned 1.0 regardless of terrain. That
behavior is preserved. The new behavior is implicit — a road that
happens to also have a river simply uses the road's 1.0 (since the
road branch is checked first), and the water-terrain check below
isn't reached. So the calc change is literally zero new logic IF the
existing road-bonus check was already first.

**Verify the existing `modifierFor` checks for road BEFORE checking
the terrain.** If it doesn't (i.e. if terrain is checked first and
`return 0` for water happens before the road check), then update the
order so the road check wins. This is the entire "bridge calc"
change.

If the existing implementation already has road-first ordering, the
travel calc change for this slice is a no-op — only document that
the bridge semantics are now intentional, not accidental.

### Test checklist

- Reload the app. All previously-painted overlays still render.
- Roads now visually curve gently rather than jagged-spoke.
- Rivers curve more pronounced than roads (rivers feel "natural,"
  roads feel "engineered").
- A hex with one road and one river on the SAME edge: both curves
  visible side-by-side, neither hides the other.
- A hex with one road and one river on different edges: each renders
  as expected.
- Drag a road from hex to hex across 5 hexes: the curve appears
  continuous from one hex to the next at every shared edge.
- Same with a river.
- Same with a route.
- Mark a route through hexes that already have roads/rivers — all
  three render, with routes always on top, roads above rivers.
- Pick A and B for travel calc on a path that crosses a hex with
  both road AND river overlays AND water terrain — the path is
  passable (1.0 modifier), not impassable. Days computes normally
  for that hex.
- Pick A and B on a path that crosses pure water (no road overlay)
  — still impassable, still triggers warning.
- Pick A and B on a path through mountain hexes with road overlays
  — road bonus still applies (1.0), days computed at fast pace.
- Hex with NO overlays renders cleanly (no stray curve artifacts).
- Reload and confirm: same hex, same edges, same overlay types, same
  curve shape (deterministic — `hashString` seeding works).
- Open a hex's detail panel, toggle road edges via the checkboxes,
  confirm the rendered curves update correctly.
- Per-type offsets are visually consistent: rivers and roads on the
  same edge across multiple hexes consistently appear on opposite
  sides.

### `docs/data-model.md` updates

No schema change. Add a brief note to the rendering section (if it
exists; otherwise add one):

```
Overlay rendering:
- Spokes are quadratic Bezier curves from hex center to a
  per-type-offset point near the edge midpoint, NOT straight lines.
- Per-type tangent offsets (rivers, roads, routes) prevent overlap
  on shared edges.
- Render order (bottom to top): terrain → rivers → roads → routes →
  path-highlight.
- Curve "bow" is small (0.08 of spoke length for roads, 0.18 for
  rivers) and seeded deterministically per (hex, edge, type) so
  redraws are identical.
- Bridge semantics: a hex with both a road and a river overlay is
  passable for travel calc purposes — road's modifier (×1.0) wins.
```

### `ROADMAP.md`

Move "Slice 14.7: Smooth overlay rendering" from Planned to Done.
(If not on the roadmap, add it under Done.) Update the header.

### Completion summary

Report:
- Total lines changed in `index.html` (added/modified/removed).
- Whether the existing `modifierFor` already had road-first ordering
  (i.e. whether the bridge semantic is a no-op or an actual change).
- Any decisions on bow magnitudes if you tuned them differently from
  the suggested 0.08 / 0.18.
- Confirmation that slice 14.7 is complete.
