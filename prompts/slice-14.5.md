# Slice 14.5: Travel time calculator

## How to work this file

This slice has two stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation before starting the next stage.

Do NOT proceed past a stage without explicit confirmation. Do NOT
rewrite `index.html` wholesale.

## Slice context

Add a travel time calculator on the map. Click hex A, click hex B,
provide a Move score, get distance + days estimate. Used to answer
"how long until the party reaches this quest location?"

Stage 2 also adds a "Mark route on map" feature — converting the
calculated path into a persistent overlay on the map (treasure-map
style red dashed line), which requires a small schema bump to add
`"route"` as a third Overlay type.

**Stage 1 is schema v10 only — no backup needed.**
**Back up `wm_unified_v1` before starting Stage 2** (schema v10 → v11).

### The math (GURPS-derived)

A 16-hour travel day at GURPS Move M covers:

```
miles_per_day = M × 4 × terrain_modifier
```

At 5 miles per hex (flat-to-flat):

```
hexes_per_day = (M × 4 × terrain_modifier) / 5
days_per_hex  = 5 / (M × 4 × terrain_modifier) = 1.25 / (M × terrain_modifier)
```

Total days for a path is the sum of per-hex days over each hex on the
straight-line route.

### Terrain modifiers

| Terrain                                          | Modifier |
|--------------------------------------------------|----------|
| plains, settled                                  | 1.0      |
| hills, forest                                    | 0.5      |
| mountain, swamp, desert, dark-forest, ruins      | 0.25     |
| water                                            | impassable (block path with warning) |
| unknown                                          | 0.5 (assume average) |

A hex with a road overlay is treated as `1.0` regardless of underlying
terrain (roads negate terrain penalties — GURPS treats them as ideal).
This applies per-hex independently; the road network doesn't need to
form a continuous path between A and B for individual road hexes to
get the bonus.

Sanity check: at Move 5 on plains, that's 4 hexes/day. Hills/forest
gives 2 hexes/day. Mountains gives 1 hex/day. Matches the expected
output.

### In scope

- Toolbar button "🧮 Travel Time" on the map.
- Click hex A (highlighted), click hex B (highlighted), panel appears.
- Path: hex line between A and B (standard hex-line algorithm).
- Per-hex breakdown: terrain, modifier, days contributed.
- Total: distance in hexes, total days (rounded to 0.5), party Move
  input (default 5, persisted in `state.ui`).
- Path visualization: highlighted hexes between A and B (ephemeral
  while panel is open).
- Per-hex manual override: dropdown to override the modifier for a
  single hex if the auto-trace got it wrong.
- Water-on-path warning: if the path crosses water, surface a warning
  but still show the result with water hexes shown as impassable —
  the user can then pick A and B differently or adjust manually.
- **"Mark route on map" feature** (stage 2): button in the calculator
  panel that persists the current path as a red dashed-line overlay
  on the map, like a treasure-map trail. Routes are a third overlay
  type alongside river and road. Multiple routes can coexist. New
  toolbar mode "Erase Routes" parallels the existing erase modes.

### Out of scope

- Pathfinding around water / mountains. The path is always straight
  hex-line.
- Saving travel calculations beyond the route overlay itself.
- Multi-segment trips (A → B → C). One leg at a time.
- Encumbrance, weather, mounted travel, magical transport. The user
  enters whatever Move score reflects current conditions.
- Forced march / pushed pace rules.
- Per-route metadata (name, party, date traveled). Routes are just
  visual markers; if you need annotation, that's a separate feature.

### State additions

In `state.ui`:

```js
mapMode: ... | 'rulerA' | 'rulerB',  // extends the slice 14 mapMode union
rulerHexA:  string | null,    // "col,row" of first picked hex
rulerHexB:  string | null,    // "col,row" of second picked hex
rulerMove:  number,           // party Move score (default 5)
rulerOverrides: { [hexKey]: 'plains'|'hills'|...|null },  // per-hex modifier overrides
```

Defensive defaults in `load()`. No version bump.

### Naming / CSS conventions

- CSS class prefix for ruler-tool styles: `r-*` (e.g. `.r-panel`,
  `.r-row`, `.r-hex-row`, `.r-warning`).
- Highlight color for picked endpoints: bright gold (`--gold` from
  the existing palette), thicker stroke than terrain hexes.
- Highlight for path hexes (excluding endpoints): same gold but
  semi-transparent fill.

---

## Stage 1 — Pick hexes, compute distance, show panel

Goal: clicking the ruler tool button enters ruler mode. Click a hex →
A is set, hex highlights. Click another hex → B is set, panel
appears with distance, terrain breakdown, days estimate. Move score
input works.

### What to add

1. **Toolbar button**: "🧮 Travel Time" next to the slice-14 mode
   buttons. Clicking enters ruler mode (`mapMode = 'rulerA'`).
   Mutually exclusive with terrain paint and overlay paint/erase
   modes.

2. **Click handlers in ruler mode**:
   - `mapMode === 'rulerA'`: clicking a hex sets `rulerHexA = key`,
     transitions to `mapMode = 'rulerB'`. Highlight A in gold.
   - `mapMode === 'rulerB'`: clicking a hex sets `rulerHexB = key`,
     keeps mode at `'rulerB'` (so user can keep clicking to change
     B). Compute and show the result panel.
   - Clicking the toolbar button while in ruler mode (rulerA or
     rulerB) exits ruler mode and clears `rulerHexA`/`rulerHexB`
     and the panel.

3. **Hex-line algorithm**: standard cube-coord interpolation.
   Pseudocode:
   ```js
   function hexLine(a, b) {
     // a, b are { col, row }; convert each to cube coords
     const A = axialToCube(a), B = axialToCube(b);
     const N = cubeDistance(A, B);
     const result = [];
     for (let i = 0; i <= N; i++) {
       const t = N === 0 ? 0 : i / N;
       const lerped = cubeLerp(A, B, t);
       const rounded = cubeRound(lerped);
       result.push(cubeToAxial(rounded));   // back to {col, row}
     }
     return result;
   }
   ```
   For axial-to-cube, use `x = col`, `z = row - (col - (col & 1)) / 2`,
   `y = -x - z` (offset coords for pointy-top with odd-r or odd-q;
   match whatever convention slice 9 uses — derive cube conversions
   from there). If the slice 9 grid is offset coords, the conversion
   here must round-trip.

4. **Per-hex modifier**:
   ```js
   function modifierFor(hexKey) {
     const h = state.hexes[hexKey];
     if (!h) return 0.5;  // off-map / unknown — assume average
     // Road bonus
     if (h.overlays && h.overlays.some(o => o.type === 'road')) {
       return 1.0;
     }
     // Terrain
     switch (h.terrain) {
       case 'plains': case 'settled':                             return 1.0;
       case 'hills':  case 'forest':                              return 0.5;
       case 'mountain': case 'swamp': case 'desert':
       case 'dark-forest': case 'ruins':                          return 0.25;
       case 'water':                                              return 0;  // impassable
       case 'unknown': default:                                   return 0.5;
     }
   }
   ```
   The `unknown` default of 0.5 is intentional — best guess for
   unexplored territory.

5. **Days computation**:
   ```js
   function daysForPath(path, move) {
     let days = 0;
     let waterHexes = 0;
     path.forEach(key => {
       const m = modifierFor(key);
       if (m === 0) { waterHexes++; return; }
       days += 1.25 / (move * m);
     });
     return { days, waterHexes };
   }
   ```

6. **Result panel**:
   - Floating div (CSS class `.r-panel`) anchored to the map area.
     Positioned over the map at a reasonable corner — top-right is
     fine.
   - Contents:
     - "From: <A label>" / "To: <B label>" header. Hex labels: hex
       name if set, else `(col, row)`.
     - Move input (number, default `state.ui.rulerMove`, range
       1–20). Changing it updates the calculation live.
     - Distance: `<N> hexes (<N×5> miles)`.
     - Total days: `<N>` rounded to nearest 0.5. Show "—" if path
       includes water.
     - Per-hex breakdown table:
       ```
       Hex      Terrain       Modifier   Days
       (1,2)    plains          1.0      0.25
       (2,2)    forest          0.5      0.50
       (3,3)    mountain        0.25     1.00
       ```
     - Close button (top-right ✕) — clears `rulerHexA`/`rulerHexB`,
       hides panel, exits ruler mode.

7. **Visual feedback on the map**:
   - A and B hexes: bright gold border, ~3px stroke.
   - No path highlight yet — that's stage 2.

### Constraints

- Move input change does NOT reset hex selection. Only the close
  button does that.
- Panel position is static (just a fixed corner). Don't make it
  draggable; that's polish.
- If user clicks A and B on the same hex, distance is 0 hexes, days
  is 0, panel still shows.
- Hex line goes through the midpoint of any tied roundings — the
  cube-round function handles the standard tie-break (round toward
  the dimension with the largest fractional component). Stick with
  the standard implementation.

### Test checklist for Stage 1

- Click "Travel Time" button. Cursor changes to crosshair (or similar).
- Click a hex — gold border highlights it.
- Click another hex — second highlight + panel appears.
- Panel shows distance in hexes and miles.
- Set move to 5, plains-only path: result matches `distance / 4`
  days.
- Set move to 5, mixed terrain: per-hex breakdown sums correctly.
- Change Move to 10 — calculation updates live without re-clicking.
- Click on a third hex — B updates, panel updates.
- Close panel — exits ruler mode, highlights clear.
- Click button to re-enter ruler mode — fresh state, no stale A/B.
- Pick endpoints with a road hex on the path — that hex's modifier
  shows 1.0 even if the underlying terrain is mountain.
- `state.ui.rulerMove` persists across reload.

---

## Stage 2 — Schema v11, route overlays, path highlight, manual overrides, water warning, polish, docs

Goal: visual path highlight while panel is open; per-hex modifier
overrides; water warning; **route overlay system** ("Mark route on
map" button + Erase Routes toolbar mode + red dashed rendering);
schema bump v10 → v11; doc updates.

**Backup `wm_unified_v1` before starting this stage.**

### Schema change

Extend `Overlay.type` union to include `"route"`:

```js
{
  type:     "river" | "road" | "route",   // NEW: "route"
  edges:    [number],
  flowFrom: number | null,
  surface:  "dirt" | "gravel" | "stone",   // ignored on routes
}
```

Routes use `edges` the same way roads/rivers do. `flowFrom` and
`surface` are unused on routes — leave them at `null` / `"dirt"` (or
omit at write time; reader code must handle missing).

### What to add

1. **Schema bump v10 → v11**:
   - Change `schemaVersion: 10` to `schemaVersion: 11` everywhere.
   - In `load()`, add `if (v < 11) migrateToV11(parsed);` to the
     migration chain.
   - Migration body is a no-op:
     ```js
     function migrateToV11(parsed) {
       // No data migration needed — routes are additive and no
       // existing overlays use type "route". Version bump only.
     }
     ```
     The function exists for parallel structure with previous
     migrations.

2. **Path highlight (ephemeral, panel-open only)**:
   - Hexes between A and B (exclusive of endpoints) get a
     semi-transparent gold fill while the panel is open.
   - Re-renders whenever A, B, or overrides change.
   - Distinct from the route overlay rendering — the gold fill is the
     "preview", the red dashes are the "marked route".

3. **Per-hex manual overrides**:
   - In the per-hex breakdown table, add an "Override" column with a
     small dropdown per row:
     - "(auto)" — uses `modifierFor()` result.
     - "Plains/road (×1.0)"
     - "Hills/forest (×0.5)"
     - "Mountain/swamp/etc (×0.25)"
     - "Impassable" (treat as water for that hex)
   - Selecting an override updates `state.ui.rulerOverrides[hexKey]`
     and recalculates.
   - Selecting "(auto)" clears the override (deletes the key).
   - Overrides are ephemeral — clear `rulerOverrides` when the panel
     closes.

4. **Water warning**:
   - If any hex on the path has modifier 0 (water, or override =
     impassable), show a warning at the top of the panel:
     "⚠ Path crosses N impassable hex(es). Total days excludes
     impassable hexes; party will need a workaround (boat, ford,
     bridge, detour)."
   - Total days line shows the sum of passable hexes only, with a
     small "+ N impassable hex(es)" caveat.

5. **"Mark route on map" button**:
   - Button in the calculator panel labeled "Mark route on map".
   - On click: walk the current hex line and add a `"route"` overlay
     to each hex along the path. For each consecutive pair `(A, B)`
     of hexes:
     - Compute the shared edge index `eA` (in A, facing B) and `eB`
       (in B, facing A). Use the same adjacency math as slice 14
       stage 2's drag-trace.
     - On A's route overlay (find or create), add `eA` if not
       present.
     - On B's route overlay, add `eB` if not present.
     - For the start hex: only the exit edge.
     - For the end hex: only the entry edge.
     - For middle hexes: both entry and exit.
   - After marking, save and re-render. The panel stays open.
   - The button can be clicked multiple times harmlessly — the
     edge-already-present check makes it idempotent for the same
     A/B selection.

6. **Erase Routes mode**:
   - New toolbar button "Erase Routes" — same parallel structure as
     "Erase Roads" / "Erase Rivers" from slice 14.
   - Click in a hex (or drag-trace through hexes) → remove the route
     overlay from each affected hex entirely.
   - Mutually exclusive with all other paint/erase/ruler modes.

7. **Route rendering**:
   - In the overlay renderer, route overlays render as red dashed
     lines from hex center to each edge midpoint:
     - Color: `#a02828` (treasure-map red).
     - Width: 2.5px (slightly thicker than roads for visibility).
     - Stroke pattern: dashed, e.g. `stroke-dasharray: "5,4"`.
   - Routes render ABOVE rivers and roads but below the
     ephemeral path-highlight gold fill. Z-order:
     terrain → roads → rivers → routes → path-highlight → pin
     layer (when slice 15 ships).

8. **Days rounding clarity**:
   - Round to nearest 0.5 days for display.
   - If raw value is 0.1 or less, show "<½ day".
   - If raw value is between 0.1 and 0.75, show "½ day" (or "1 day"
     above 0.75).
   - For ≥1 day, show "N days" or "N.5 days" (rounded to nearest 0.5).

9. **`docs/data-model.md` updates**:
   - Bump every reference to v10 → v11 in the doc.
   - Update the `Overlay.type` union to include `"route"`. Add a
     short note: "Routes are visually distinct (red dashed) and
     ignore `flowFrom` and `surface`. They're typically created via
     the travel calculator's 'Mark route on map' button but can also
     be manually erased via the Erase Routes toolbar mode."
   - Add a migration note: "Migration v10 → v11: no-op. The
     `Overlay.type` union gained `"route"`; existing overlays remain
     valid with no data change."
   - Add the new `state.ui` ruler fields:
     ```js
     mapMode:        ... | 'rulerA' | 'rulerB' | 'eraseRoute',
     rulerHexA:      string | null,
     rulerHexB:      string | null,
     rulerMove:      number,
     rulerOverrides: { [hexKey]: string | null },
     ```
   - Note that `rulerOverrides` is ephemeral and clears on panel
     close.

10. **`ROADMAP.md`**: move "Slice 14.5: Travel time calculator helper"
    from Planned to Done. Update the header.

### Constraints

- "Mark route on map" does NOT close the panel — the user might want
  to mark, then adjust A/B, then mark again.
- Marking the same path twice is idempotent (no duplicate edges).
- The marked route persists across reload — it's a real overlay.
- "Erase Routes" mode does not affect roads or rivers.
- Closing the calculator panel clears the gold preview highlight but
  leaves any marked routes intact.
- Routes don't grant the road bonus in `modifierFor()`. Routes are
  just visual markers, not roads. Only `type === "road"` overlays
  trigger the ×1.0 bonus.

### Test checklist for Stage 2

- Reload after deploy. State loads cleanly with `schemaVersion: 11`.
- Pick A and B. Path hexes between them are highlighted in
  semi-transparent gold (preview).
- Click "Mark route on map" — red dashed line appears along the
  path. Reload — route persists.
- Pick a different A/B and mark a second route — both routes coexist
  on the map.
- Switch to "Erase Routes" mode. Click on a hex with a route — that
  hex's route segment disappears (other hexes' route segments
  remain).
- Drag across multiple hexes in Erase Routes mode — sweeps the
  route(s) clean across that swath.
- Roads and rivers are unaffected by Erase Routes.
- Mark a route through a hex that already has a road — both render
  (road in tan, route in red dashes, both visible).
- Override one hex's modifier from auto (mountain, ×0.25) to ×1.0 —
  total days drops accordingly. Marking the route after override
  still uses the auto-traced hex line (overrides only affect the
  days calculation, not the path geometry).
- Reset override to "(auto)" — total restores.
- Pick a path crossing water — warning shows, days excludes water,
  caveat appears.
- Override a non-water hex to "Impassable" — same warning logic.
- Days rounding: hand-pick values that give 0.05, 0.5, 1.0, 1.5, 2.3,
  3.7 — confirm "less than half", "half", "1", "1.5", "2.5", "3.5"
  display correctly.
- Close panel — gold preview clears; marked routes stay; overrides
  reset; reopening with the same A/B starts fresh (preview only).
- Doc diff: schema bumped, Overlay.type includes "route", migration
  note added, ui shape updated.
- ROADMAP diff: slice 14.5 in Done.
- All other tabs / tools still work.

### Completion summary to write at the end

- Total lines added to `index.html` (roughly).
- Confirmation that slice 14.5 is complete.
- Note any GURPS rule interpretations you took liberties with.
