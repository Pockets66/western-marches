# Slice 14.11 — Map undo system

Adds undo/redo for map-state mutations. Scoped to map state only
(`state.mapMeta` + `state.hexes`); other tabs are unaffected. **No schema
change**: the undo stack is in-memory and does not persist across reloads.

Originally planned as slice 14.6 (pre-segment-model), deferred and reframed
to build on the consolidated 14.10 model.

## Motivation

Currently the only recovery for a misclick on the map is page reload (which
loses unsaved work elsewhere) or restoring from a manual export. Painting,
erasing, segment edits, and especially resize are all destructive with no
recourse. Undo is the missing baseline interaction.

## Design decisions

- **Snapshot, not command pattern.** Each undoable action pushes a
  pre-mutation snapshot of `{ mapMeta, hexes }` onto a stack, serialized as
  JSON. Restoring an undo entry means parsing it back and assigning. Simpler
  than per-action invert functions, naturally handles resize (which would be
  awful to invert), and uniform across all mutation paths.
- **Scope: map state only.** The undo stack contains only `mapMeta` and
  `hexes`. Pins, factions, sessions, etc. are untouched. (Pins live in
  `state.pins`, not nested in hexes — they're outside scope until slice 15
  introduces them on the map.)
- **In-memory only.** Stack lives in module-local variables. Reload clears
  it. Not written to localStorage.
- **Stack cap: 50 entries.** When pushing onto a full stack, drop the oldest.
  Redo stack also caps at 50.
- **Drag-paint = one undo unit.** Tracing across N hexes by drag must produce
  one undo entry, not N.
- **Map import clears both stacks.** Import is a wholesale replacement;
  preserving prior history would let undo restore a state from a *different*
  map.
- **Keyboard shortcuts: Map tab only.** Ctrl/Cmd-Z and Ctrl/Cmd-Shift-Z
  (also Ctrl-Y) only fire when the Map tab is active, to avoid clashing with
  textarea undo on Sessions/Rumors/etc.
- **No undo for selection or UI state.** `_mapSelHex`, hover, ruler picks,
  current tool/brush — none of these go on the stack.

## Stages

Each stage is one logical chunk. Stop after each for a "go" confirmation.

---

### Stage 1 — Stack foundation, toolbar buttons, keyboard shortcuts

Add the core undo/redo machinery without yet wiring it to any mutation. End
of stage: buttons exist, keyboard fires, but stacks are empty because nothing
pushes yet.

**Add module-locals** near the other map state:

```js
let _mapUndoStack = [];   // each entry: JSON string of { mapMeta, hexes }
let _mapRedoStack = [];
const _MAP_UNDO_CAP = 50;
```

**Add helpers:**

- `_mapSnapshot()` → returns `JSON.stringify({ mapMeta: state.mapMeta, hexes: state.hexes })`.
- `_mapRestoreSnapshot(snap)` → parses, assigns to `state.mapMeta` and
  `state.hexes`. Calls `_invalidateOverlayCache()`, `save()`, `drawWorld()`,
  and re-renders the info panel if a hex is selected (clear selection if the
  selected hex no longer exists in the restored state).
- `_pushMapUndo()` → pushes `_mapSnapshot()` onto `_mapUndoStack`, drops
  oldest if over cap, clears `_mapRedoStack`, refreshes button enabled state.
- `mapUndo()` → if undo stack non-empty: push current snapshot to redo,
  restore from undo top.
- `mapRedo()` → mirror: if redo stack non-empty, push current to undo,
  restore from redo top.

**Toolbar buttons.** Add Undo and Redo buttons to `_buildMapToolbar()` next
to the Resize/Export/Import group. Disabled state when respective stack is
empty. A small refresh helper `_refreshUndoButtons()` toggles disabled.

**Keyboard.** Add a `keydown` handler on `document` that:
- Only fires when `state.ui.activeTab === 'map'`.
- Ignores the event if `e.target` is an `<input>` or `<textarea>` (so the
  segment editor's selects, hex name field, etc. keep their native behavior).
- `(Ctrl|Meta)+Z` without shift → `mapUndo()`.
- `(Ctrl|Meta)+Shift+Z` or `(Ctrl|Meta)+Y` → `mapRedo()`.
- `preventDefault()` on match.

**Test.**
- Buttons render disabled. Keyboard shortcut on Map tab does nothing
  (silent), shortcut on Sessions tab still does native textarea undo.

---

### Stage 2 — Wire single-action mutations

Add `_pushMapUndo()` calls before mutation in every map-state-changing
function, except drag-paint flows (those land in stage 3).

**Functions to wrap:**

- `applyMapMode(c, r)` for `paint`, `erase`, `explored` modes only.
- `updateHexName`, `updateHexNote` — but **debounced**: only push if 750ms
  has elapsed since the last push from a name/note edit on this hex, OR if
  the input has lost focus since. (Otherwise typing "Westfall" creates 8
  undo entries. We want one per "edit session".) Use a single timer +
  last-edit-key tracker.
- `setOverlaySurface`
- `setWaterOverlay`
- `segEditorAddSeg`, `segEditorRemoveSeg`, `segEditorSetFrom`,
  `segEditorSetTo`, `segEditorSetFlow`
- `segEditorSyncFlow` — push *before* the user confirm, only if confirm
  passed.
- `_eraseSingleSegmentAt` (the shift-click erase path) — see stage 3 caveat
  about whether shift+drag erase is grouped.

**Test.**
- Paint a hex, undo → terrain reverts. Redo → terrain comes back.
- Add a road segment via segment editor, undo → segment vanishes.
- Type 5 characters into hex name, wait 1 second, type 3 more: 2 undo
  entries, not 8.
- Click off the name field, then back, then type more: that's a new entry.
- Sync flow on a 5-segment river: 1 undo entry, undoes all 5 segment flow
  changes at once.
- Stack cap: do 51 paint actions, verify oldest is dropped (undo 50 times,
  the final undo restores the 1st post-cap state, not the original).

---

### Stage 3 — Drag operations as single undo units

Drag-paint and drag-erase currently emit many mutations between mousedown
and mouseup. Group each drag gesture into one undo entry.

**Mechanism.** Replace ad-hoc per-mutation pushes (which stage 2 wouldn't
have added for drag paths) with a transaction model:

- New module-local: `let _mapUndoTxnSnap = null;`
- `_beginUndoTxn()` → if `_mapUndoTxnSnap` is null, set it to
  `_mapSnapshot()`. (Idempotent within a transaction.)
- `_commitUndoTxn(mutated)` → if `_mapUndoTxnSnap` is non-null AND `mutated`
  is true: push `_mapUndoTxnSnap` to undo stack, clear redo, refresh
  buttons. Always null out `_mapUndoTxnSnap` at end.
- `_abortUndoTxn()` → null out `_mapUndoTxnSnap` without pushing. (Used if
  no mutation actually occurred, e.g. drag started but user moved zero
  pixels.)

**Wire into drag handlers:**

- `mousedown` for `paintRoad`, `paintRiver`, `eraseRoad`, `eraseRiver`,
  `eraseRoute`: call `_beginUndoTxn()`. (For erase-modes, also wrap the
  immediate single-click erase that mousedown does — that's still part of
  the "drag" transaction, just zero-length.)
- `mousemove` mutations: no change. They mutate without touching the
  transaction.
- `mouseup` and `mouseleave` (`_finishOverlayDrag`): call
  `_commitUndoTxn(_mapDragOccurred || /* erase always counts */)`.

For non-drag paths (segment editor, terrain paint single-click, etc.),
stage 2's `_pushMapUndo()` calls remain. Those don't go through
`_beginUndoTxn` / `_commitUndoTxn`.

**Test.**
- Drag a road across 6 hexes: 1 undo entry. One Ctrl-Z restores the entire
  pre-drag state.
- Click-paint a single road stub: 1 undo entry (via the mousedown→mouseup
  path with no movement). Same Ctrl-Z behavior.
- Drag-erase across 4 hexes: 1 undo entry.
- Shift+drag erase (single-segment): 1 undo entry covering all hexes
  touched.
- Mousedown then mouseup on the same hex with no movement and no
  mutation (e.g. paint mode but cursor lands on an already-existing
  identical segment): no undo entry created (transaction aborted).

---

### Stage 4 — Resize and import; final polish

**Resize.** `applyMapSettings` pushes a snapshot before mutating. The push
must happen *after* the user confirms the warning (if any), so that
canceled resizes don't pollute the stack.

**Import.** `importMap` clears both `_mapUndoStack` and `_mapRedoStack`
after successful import, before refreshing the UI. The current state cannot
be restored to a state from a different map. Refresh button disabled state.

**Status feedback.** When undo/redo fires (button or keyboard), briefly
flash the status bar with `UNDO` / `REDO`. Use the same flash mechanism as
`mapSaveNoteFlash` if it generalizes; otherwise just a 600ms text swap.

**Documentation.** Update:
- `data-model.md`: add a short "Map undo" subsection under the Hex section
  (or as its own section). Note the in-memory-only nature, the 50-entry
  cap, and the import-clears-stack rule.
- `ROADMAP.md`: move slice 14.11 from Deferred / In progress to Done.

**Edge cases to verify:**
- Undo a resize that shrunk the map: hexes outside the new bounds were
  deleted via `delete state.hexes[k]`. Snapshot captured them before
  deletion, so undo restores them. Verify.
- Undo while segment editor is open: the panel must re-render against the
  restored hex (or close cleanly if the selected hex no longer exists,
  e.g. after undoing a resize-grow that selected a then-deleted hex).
- Redo after a fresh mutation: redo stack should already be cleared by the
  intervening mutation. Verify Ctrl-Shift-Z is a no-op in that state.
- Tab switch then return: undo/redo still works against same stack.
- Undo across a save boundary: `save()` is called inside
  `_mapRestoreSnapshot`, so localStorage tracks the undone state. If user
  reloads, they see the post-undo state and the stack is empty. Document
  this behavior — it's correct, just worth being explicit about.

---

## Out of scope

- Undo for non-map state (factions, sessions, NPCs, etc.).
- Persisting the undo stack across reloads.
- Multi-tab undo coordination.
- Branching undo / undo tree / redo with multiple branches.
- Pin-aware undo (slice 15 will add pins to the map; that slice owns
  whether pin mutations are undoable).
- Visual diff or "you undid: terrain change in (3, -2)" annotations.

## Files

- `index.html` — all behavior changes.
- `data-model.md` — short subsection on map undo semantics.
- `ROADMAP.md` — move slice 14.11 to Done.

## Definition of done

- All four stages land cleanly with no regressions.
- 50+ undo entries possible; cap enforced.
- Drag operations are one undo unit each.
- Resize and import behave as specified.
- Keyboard shortcuts only fire on Map tab and not inside text inputs.
- No schema change; loading a 14.10 save and using undo Just Works.
