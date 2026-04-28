# Slice 14.6: Map undo

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

Add a per-stroke undo stack for map actions. Drag-trace painting and
erasing can clobber overlapping data quickly — a single accidental
mouse drag through 5 hexes wipes 5 hexes of overlays in one frame.
This slice adds a bounded in-memory undo stack so those mistakes are
recoverable.

This is a **pure UI slice**. No schema change, no migration, no
backup required. Schema stays at v11.

### Scope decisions (already made — don't relitigate)

- **In-memory only.** The undo stack lives in JS memory, not in
  `state` or localStorage. A page reload clears the stack. Acceptable
  trade-off — undo is for "I just did something dumb", not "yesterday
  I did something dumb".
- **Per-stroke, not per-hex.** A drag from hex A to hex E (5 hexes
  affected) is ONE undo entry, not five. A click that adds one edge
  to one hex is also one entry.
- **Map actions only.** Terrain paint, overlay paint, overlay erase,
  "Mark route on map" button. NOT covered: hex detail panel edits
  (name, notes, terrain via dropdown), faction/NPC/event/etc. edits,
  travel calculator state changes.
- **Depth limit: 20 entries.** Older entries fall off the bottom.
- **No redo.** Possibly added later if it turns out to be missed.

### Actions covered

| Action                          | Trigger                                  |
|---------------------------------|------------------------------------------|
| Terrain paint stroke            | One drag in terrain paint mode           |
| Road paint stroke               | One drag in paintRoad mode               |
| River paint stroke              | One drag in paintRiver mode              |
| Road erase stroke               | One drag in eraseRoad mode               |
| River erase stroke              | One drag in eraseRiver mode              |
| Route erase stroke              | One drag in eraseRoute mode              |
| Mark route on map               | One click of the calculator panel button |

A "stroke" includes the initial mousedown click PLUS any drag
movement until mouseup or mouseleave. A pure click (no drag) is also
a stroke of length 1.

### State additions

In `state.ui` (will be defaulted defensively in `load()`):

Nothing persisted. The undo stack is a module-level JS variable, not
part of `state`:

```js
// At the top of the script, near other module-level vars
let undoStack = [];   // newest at the END of the array
const UNDO_DEPTH = 20;
```

### Naming / CSS conventions

- CSS class prefix for undo button: `u-*` (e.g. `.u-undo-btn`,
  `.u-undo-disabled`).
- Reuse existing palette and button styles. The undo button is on
  the map toolbar alongside paint/erase mode buttons but is NOT a
  mode toggle — it's a one-shot action button.

---

## Stage 1 — Stack, undo button, paint/erase coverage

Goal: the undo button on the map toolbar reverses the most recent
paint/erase stroke. Strokes are captured atomically (one drag = one
entry). Up to 20 strokes can be undone in sequence.

### What to add

1. **Undo stack module-level state** as described above.

2. **Snapshot helper functions**:
   ```js
   function snapshotHex(hexKey) {
     const h = state.hexes[hexKey];
     if (!h) return { key: hexKey, existed: false };
     return {
       key: hexKey,
       existed: true,
       data: JSON.parse(JSON.stringify(h)),
     };
   }
   
   function restoreHexSnapshot(snap) {
     if (!snap.existed) {
       delete state.hexes[snap.key];
     } else {
       state.hexes[snap.key] = JSON.parse(JSON.stringify(snap.data));
     }
   }
   ```
   Deep clone with JSON round-trip is fine — Hex objects are plain
   data.

3. **Stroke lifecycle helpers**:
   ```js
   let activeStroke = null;
   
   function beginStroke(label) {
     activeStroke = { label, snapshots: new Map() };  // hexKey → snapshot
   }
   
   function captureHex(hexKey) {
     if (!activeStroke) return;
     if (activeStroke.snapshots.has(hexKey)) return;  // already captured
     activeStroke.snapshots.set(hexKey, snapshotHex(hexKey));
   }
   
   function endStroke() {
     if (!activeStroke) return;
     if (activeStroke.snapshots.size > 0) {
       undoStack.push(activeStroke);
       if (undoStack.length > UNDO_DEPTH) undoStack.shift();
     }
     activeStroke = null;
     renderUndoButton();
   }
   
   function abortStroke() {
     activeStroke = null;
   }
   ```
   The `captureHex` function is idempotent per stroke — only the
   FIRST snapshot of a hex within a stroke is kept. Subsequent
   captures are no-ops, so subsequent edits to the same hex within
   the same stroke don't overwrite the original "before" state.

4. **Wire snapshot capture into existing handlers**:

   For each of these actions, call `beginStroke(label)` at the
   start of the action, `captureHex(hexKey)` BEFORE mutating each
   hex, and `endStroke()` at the end:

   - **Terrain paint** — find the existing terrain paint handlers
     (mousedown, mousemove during drag, mouseup). Wrap with
     `beginStroke('terrain paint')` / `endStroke()`. Call
     `captureHex(hexKey)` for each hex BEFORE its terrain is changed.
   - **paintRoad / paintRiver / eraseRoad / eraseRiver / eraseRoute**
     modes — same pattern. Use `'paint road'`, `'erase river'`, etc.
     as labels for clarity.

   Snapshot capture must happen BEFORE the hex is mutated, not after.
   If the existing code mutates first then saves, restructure: capture
   first, mutate, save.

   On `mouseleave` from the canvas mid-drag, treat that as
   `endStroke()` — the stroke completes, even though the user dragged
   off-canvas. (If they re-enter and continue dragging, that's a new
   stroke.)

5. **"Mark route on map" button** (from slice 14.5):
   - Wrap the marking logic with `beginStroke('mark route')` /
     `endStroke()`.
   - Call `captureHex(hexKey)` for each hex along the path BEFORE
     adding the route overlay edges.

6. **Undo button on the map toolbar**:
   - Label: `↶ Undo`. Tooltip: "Undo last map action (Ctrl+Z)".
   - Position: leftmost on the map toolbar, separated from the mode
     buttons by a small gap. NOT a mode toggle — single click is a
     one-shot action.
   - Disabled state: when `undoStack.length === 0`, button is grayed
     out and unclickable.

7. **Undo handler**:
   ```js
   function performUndo() {
     if (undoStack.length === 0) return;
     const stroke = undoStack.pop();
     stroke.snapshots.forEach(snap => restoreHexSnapshot(snap));
     save();
     renderMap();        // or whatever the existing render call is
     renderUndoButton();
     // If a hex detail panel is open for an affected hex, re-render it
     if (state.ui.activeHexKey && stroke.snapshots.has(state.ui.activeHexKey)) {
       renderHexDetail(state.ui.activeHexKey);  // adjust to actual function name
     }
   }
   ```

8. **`renderUndoButton()` helper** updates the disabled/enabled
   state of the button based on `undoStack.length`. Call this after
   every `endStroke()` and after every `performUndo()`.

### Constraints

- The undo stack is module-level, not part of `state`. Don't try to
  persist it.
- A stroke with no actual mutations (e.g. the user clicked in paint
  mode but the edge was already present) produces a snapshots map
  of size 0 — `endStroke()` won't push it. The undo button stays
  unchanged.
- Consecutive identical strokes (e.g. paint, paint, paint same hex)
  each get their own undo entry. Don't try to coalesce — depth 20
  is plenty.
- If a stroke is already active when another `beginStroke()` is
  called (shouldn't happen, but defensive): call `endStroke()` first
  on the existing stroke, then start the new one. Don't drop the
  existing stroke silently.

### Test checklist for Stage 1

- Click "Paint Roads", drag through 5 hexes. Click ↶ Undo — all 5
  hexes' roads are removed (back to whatever they had before).
- Drag a road through 3 hexes, drag another road through 3 different
  hexes, undo. Only the second drag is reversed.
- Undo again — first drag reversed.
- Undo again with empty stack — button is disabled, no-op.
- Paint terrain on 4 hexes (one drag) → undo → all 4 revert.
- Paint road on hex A, paint different road on hex A again (so the
  edge list grew) → undo → hex A is back to ONLY the first paint
  state, not erased entirely.
- Drag-erase rivers through 6 hexes → undo → all 6 rivers restored.
- "Mark route on map" through a 7-hex path → undo → all 7 hexes
  lose their route overlay segments.
- Hex detail panel open on hex X. Paint a road on hex X via the
  toolbar (different hex). Then paint on hex X. Undo. The hex X
  detail panel re-renders to show the reverted state.
- Hard reload — undo stack clears. ↶ Undo button is disabled.
- Quickly do 25 paint strokes — only the most recent 20 are
  undoable. The first 5 are dropped silently.

---

## Stage 2 — Keyboard shortcut, polish, docs

Goal: Ctrl+Z / Cmd+Z works as expected. Edge cases polished. Docs
reflect the new behavior.

### What to add

1. **Keyboard shortcut**:
   - Listen on `document` for `keydown`.
   - Trigger `performUndo()` when:
     - `(e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey`
     - AND `e.target` is NOT an `<input>`, `<textarea>`, or
       `[contenteditable="true"]` element. (Ctrl+Z inside text fields
       must do its native browser undo.)
     - AND the map tab is active. Don't trigger undo on the Factions
       tab, even if the map's stack has entries.
   - `e.preventDefault()` only when actually handling — otherwise pass
     through.

2. **Visual feedback on undo**:
   - When `performUndo()` runs, briefly flash the undo button (e.g.
     ~150ms scale or background pulse). Use a small CSS animation,
     keep it subtle.
   - This makes the keyboard shortcut feel responsive even though
     the user can't see a click happen.

3. **Tooltip refinement**:
   - When the stack is non-empty, the tooltip should show the label
     of the action that will be undone:
     `"Undo: paint road (Ctrl+Z)"`
   - Get the label from `undoStack[undoStack.length - 1].label`.

4. **Defensive ID-handling**:
   - If a snapshot's hex was deleted between snapshot and undo (no
     current path does this, but future code might), the
     `restoreHexSnapshot` re-creates it from the snapshot data.
     `existed: true` snapshots restore the hex; `existed: false`
     snapshots delete it. This is already correct in the helper —
     just verify by inspection.

5. **`docs/data-model.md` updates**:
   - No schema change. Schema stays at v11.
   - Add a brief note in the doc near the `state.ui` section:
     ```
     The map undo stack (`undoStack` in module scope, depth 20) is
     in-memory only and not part of persisted state. Map actions
     covered: terrain paint stroke, overlay paint/erase stroke,
     "Mark route on map". Hard reload clears the stack.
     ```

6. **`ROADMAP.md`**: move "Slice 14.6: Map undo" from Planned to
   Done. Update the header.

### Constraints

- Don't add Ctrl+Y or Cmd+Shift+Z for redo. There's no redo this
  slice.
- Keyboard handler must NOT fire when typing in text fields. Test
  this explicitly — it's the most common bug in this kind of feature.
- The flash animation must not cause layout shift. Animate transform
  or background-color, not width/height/margin.

### Test checklist for Stage 2

- Press Ctrl+Z (or Cmd+Z on Mac) on the map tab after a paint stroke
  — undo fires.
- Press Ctrl+Z while focused on the hex name input field — native
  text undo fires, NOT map undo.
- Press Ctrl+Z while focused on a textarea (e.g. event description) —
  native text undo fires, NOT map undo.
- Switch to Factions tab. Press Ctrl+Z. Nothing happens (or native
  text undo if focused on a text field).
- Tooltip on undo button shows the next action's label.
- Click ↶ Undo button. Visual flash plays.
- Press Ctrl+Z. Same flash plays.
- Doc diff: data-model.md note added; ROADMAP.md updated.
- All other tabs and tools still work.
- The route overlay drag-erasers, road erasers, and river erasers
  all still work and are all individually undoable.

### Completion summary to write at the end

- Total lines added to `index.html` (roughly).
- Confirmation that slice 14.6 is complete.
- Note any actions you wrapped that weren't on the list above (and
  why).
