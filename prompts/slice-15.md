# Slice 15 — Local maps + pins

Surfaces the pin entity (already in the schema as `state.pins: [Pin]` since
slice 12, but with no UI) and adds a new "local map" feature: a per-hex
detailed view backed by an optional uploaded image, on which pins are placed
at specific coordinates.

**Schema bump: v13 → v14.** Adds `state.localMaps: { [hexKey]: LocalMap }`.
Pin schema unchanged from what's already documented.

## Motivation

Western Marches play accumulates locations: settlements, ruins, dungeons,
threats, camps, landmarks. The world hex map captures *which hex* something
is in, but a 24-mile hex is too coarse for detail like "the inn is northwest
of the temple, the docks are at the southern edge." Local maps give the GM
a place to capture that detail: optionally drop in a sketched/exported map
image and pin specific locations onto it.

Pins are also the connective tissue for slices 17–18 — NPCs, rumors, and
quests already reference pins by `pinId` in the schema, but those references
have no UI to manage on the pin side until pins themselves have a UI. This
slice creates that surface.

## Design decisions

- **One local map per hex.** Keyed by `hexKey`. For multiple locations in
  one hex, place multiple pins on the same map.
- **Local maps are optional.** A hex can have pins without ever having a
  local map. In that case, pins still render on the world map (as dots in
  their hex), and `(x, y)` coords default to the local map's center if/when
  one is added later.
- **Images embed as data URLs in localStorage.** On upload: resize to max
  1600px on longest dimension, re-encode as JPEG quality 0.85. Documented
  user-facing in the upload control. (IndexedDB-backed image store is
  deferred — would justify its own slice.)
- **Pin ↔ everything-else cross-links are out of scope.** This slice gives
  pins CRUD and a faction selector; NPC/rumor/quest linkages reveal in
  slices 17 and 18.
- **Undo extends to pins and local maps.** Slice 14.11 deferred this
  decision to slice 15. Widen `_mapSnapshot()` to capture
  `{ mapMeta, hexes, pins, localMaps }`. All four restore atomically. Same
  50-entry cap, same in-memory-only semantics.
- **Discovered/undiscovered.** The Pin schema's `discovered: boolean` is
  rendered (faded ring for undiscovered) but no separate "player view"
  toggle — that's a future slice.
- **Pin glyphs by type, color by faction (or type fallback).** A single
  uppercase letter per type rendered inside a small filled circle. Faction
  color used if `factionId` is set, else a per-type default color.

## Stages

Each stage is one logical chunk. Stop after each for "go" confirmation.

---

### Stage 1 — Schema v14 + pin rendering on world map

**Schema migration v13 → v14.** Add `migrateToV14()`:

- `state.localMaps = state.localMaps || {}`. Defensive only.
- `state.pins = state.pins || []`. Should already exist but belt-and-suspenders.
- For each pin in `state.pins`: ensure all fields conform to the documented
  shape. Missing fields default per schema (`discovered: false`,
  `factionId: null`, `notes: ''`, `type: 'other'`, `x: 0`, `y: 0`).
- Increment `schemaVersion` to 14.

**LocalMap shape** (add to `data-model.md`):

```js
// Stored at state.localMaps[hexKey]
{
  imageDataUrl: string | null,   // base64 JPEG data URL, or null for blank canvas
  width:  number,                 // canvas width in pixels
  height: number,                 // canvas height in pixels
}
```

When `imageDataUrl` is null, the canvas renders blank (parchment-tinted)
at the stored width/height. Default size for a blank local map: 800×600.

**Constants and helpers in `index.html`.**

- `PIN_TYPES`: array of `{ key, label, glyph, color }` for the 7 types
  (settlement S/gold, dungeon D/dark-purple, ruin R/grey, landmark L/teal,
  threat T/red, camp C/orange, other O/text-dim).
- `pinDisplayColor(p)`: returns faction color if `p.factionId` is set and
  that faction exists, else the per-type color from `PIN_TYPES`.
- `_pinsByHex()`: memoized `{ [hexKey]: [Pin] }` map. Invalidate on any pin
  mutation (add to existing `_invalidateOverlayCache`-style invalidation, or
  add a sibling `_invalidatePinCache` and call it where needed).

**World map rendering.** In `drawWorld()` after overlays render and before
ruler/selection chrome:

- For each hex with pins: draw small filled circles (radius = `hexSize *
  0.12`) clustered around the hex center. 1 pin: dead-center. 2 pins:
  side-by-side. 3+: hex-pattern around center within a `hexSize * 0.3`
  radius. (Don't over-engineer — visible-and-non-overlapping is the bar.)
- Each pin dot: filled with `pinDisplayColor(pin)`, 1px dark stroke, single
  uppercase glyph from `PIN_TYPES` rendered in white at small font size
  (~`hexSize * 0.18` px).
- Undiscovered pins: dashed stroke instead of solid.
- Skip rendering pins on hexes with `terrain: 'unknown'` (fog of war is
  cleaner that way; if user wants a pin shown they should reveal the hex).

**Hex info panel.** Add a "Pins" section above the overlay editor. Lists
pins in this hex, each row showing:
- Type glyph + name (or `<unnamed>` if blank)
- Discovered toggle (eye icon, on/off)
- Edit button → opens an inline editor (name, type select, faction select
  with FACTION_COLORS, notes textarea, delete button)
- "+ New pin" button at the bottom of the section → creates a pin centered
  on the local map (or `(400, 300)` if no map yet), prompts inline for name.

**Test.**
- Empty state: hex with no pins shows "No pins" placeholder. Add a pin via
  the panel, name it, type "settlement". A gold dot with `S` glyph appears
  on that hex on the world map.
- Faction assignment: assign the pin to a faction. Dot recolors to the
  faction color.
- Toggle discovered off: pin dot's stroke becomes dashed.
- Add 4 pins to one hex: all 4 visible, none overlapping, all roughly
  centered.
- Delete a pin: dot disappears, panel row gone.
- Hex with pins becomes `unknown` terrain: pins still in data, dots not
  rendered. Repaint terrain → dots return.
- Save and reload: pins persist (localStorage round-trip).
- v13 → v14 migration: load a slice-14.11 save, verify schemaVersion
  becomes 14, no pins present unless previously authored, no errors.

---

### Stage 2 — Local map modal: open, render, image upload

End of stage: a hex with pins can open a modal showing the local map
(blank or imaged) with pins rendered at their `(x, y)` coords. No
interaction with pins on the local map yet — just rendering.

**Modal shell.** Add a full-screen overlay div `#localmap-modal` (hidden by
default). Header bar with hex name (or `(col, row)`), close button (×). Body
contains the local map canvas inside a scrollable container. Footer/toolbar
area for image controls (Upload Image, Remove Image) and pin tool buttons
(stage 3 fills these in).

**Open flow.** Hex info panel's pin section gets a button: "Open Local Map"
if `state.localMaps[hexKey]` exists, else "Create Local Map" (which creates
a default 800×600 blank entry, then opens). Button is enabled regardless of
whether the hex has pins — you can have a local map with no pins.

**Render.** Modal body has its own `<canvas id="localmap-canvas">`:

- Sized to the LocalMap's `width × height`.
- If `imageDataUrl` is non-null, draw the image (cached `Image` object).
- Else, fill with a parchment tint (`var(--panel-mid)` or similar).
- Draw all pins for this hex at their `(x, y)`. Pin appearance same scheme
  as world map (colored circle + glyph), but rendered larger (radius 14px),
  with the pin's name as a label below the circle.
- Undiscovered pins: dashed stroke.

**Image upload.** Toolbar's "Upload Image" button triggers a file input.
On file selection:
- Load file into an `Image` via `URL.createObjectURL` (don't keep the
  blob).
- Compute target size: scale longest dimension to ≤1600px, preserving
  aspect ratio.
- Render to an offscreen canvas at target size, export via `toDataURL(
  'image/jpeg', 0.85)`.
- Save to `state.localMaps[hexKey].imageDataUrl`, update `width`/`height`
  to the resized dimensions.
- Push undo entry. Re-render canvas.
- Show a toast: "Image stored (~NkB)".

User-facing note in the upload tooltip: "Images resized to 1600px max,
JPEG quality 85%. Stored locally; total budget ~5MB."

**Remove image.** "Remove Image" button → confirm prompt → set
`imageDataUrl = null`, push undo, re-render. The local map and its pins
stay; only the background image is cleared.

**Close.** × button or Escape key closes the modal. World map view stays
on the same hex selection.

**Test.**
- Hex with no local map: pin section shows "Create Local Map" button.
- Click it: blank 800×600 canvas opens; any existing pins render at their
  `(x, y)` (which is `(400, 300)` for new ones).
- Upload a 3000×2000 JPG: resized to 1600×1067, displayed correctly.
- Upload a 4MB PNG: data URL is much smaller (likely <300KB) after JPEG
  re-encode. Save and reload — image persists.
- Remove image: blank canvas, pins remain.
- Close modal, reopen: state preserved.
- Escape key closes.
- Open from hex A, close, open from hex B: hex B's local map renders
  (not A's stale state).
- Undo after image upload: prior image (or blank) restored.

---

### Stage 3 — Local map interaction: add, drag, edit pins

Pin manipulation directly on the local map.

**Add pin tool.** Toolbar mode button "Add Pin" (icon: pin glyph). When
active, clicking on the canvas:
- Creates a pin at click `(x, y)` with default type `'other'`, blank name,
  `discovered: false`, `factionId: null`, `notes: ''`.
- Pushes undo entry.
- Selects the new pin and opens the pin editor (a small floating panel
  near the pin, see below).
- Stays in Add Pin mode for rapid placement, until user clicks the tool
  off or hits Escape.

**Select / drag.** Default tool (Select). Click on a pin to select.
Mouse-down + drag updates `(x, y)` live; mouseup commits and pushes a
single undo entry for the drag (use the same transaction model as
slice 14.11). Click on empty canvas deselects.

**Pin editor panel.** Floats near the selected pin (or anchors to a side
of the modal if the pin is near a screen edge). Contents:
- Name input
- Type select (7 options)
- Faction select (None + all factions with color swatches)
- Notes textarea
- Discovered toggle
- Delete button (confirms first)

All edits push undo entries with the same debounce model as 14.11 for
text inputs (750ms debounce on name/notes, immediate on selects/toggles).

**Cursor.** Pin hover shows pointer cursor. Drag shows grabbing cursor.
Add Pin mode shows crosshair.

**Z-ordering.** Pins render on top of the image. Selected pin renders on
top of unselected. (No pin layers / grouping in v1.)

**Sync to world map.** After any pin mutation in the local map (add,
move, edit, delete), the world map's pin dots also update. The shared
pin store + cache invalidation handles this — just verify it works after
closing the modal.

**Test.**
- Add 3 pins by clicking. Each opens the editor. Name them, set types.
  Close modal. World map shows 3 dots in that hex.
- Drag a pin halfway across the canvas. Release. Reopen — pin is at the
  new position. One undo restores the original position.
- Drag with no movement (mousedown→mouseup at same spot): no undo entry
  (transaction abort).
- Edit a pin's name "T" → "Tower of Skulls" via 750ms debounce: one undo
  entry, not seven.
- Toggle discovered: undo entry added immediately.
- Delete a pin via editor: undo restores it (with all its fields).
- Click off pin: editor closes; pin deselected.
- Add Pin mode + Escape: returns to Select mode.

---

### Stage 4 — Polish and edge cases

**Undo integration.** Verify `_mapSnapshot()` and `_mapRestoreSnapshot()`
now cover `pins` and `localMaps` alongside `mapMeta` and `hexes`:
- After a pin mutation outside the local map modal (e.g. via hex info
  panel), keyboard Ctrl-Z works as expected.
- After a local map image upload, Ctrl-Z restores prior image state.
- Across mixed operations (paint hex, add pin, paint another hex, drag
  pin): linear undo history walks back through all of them in order.

**Cascade rules.** Update `data-model.md` and implement:
- **Delete faction** → also set `factionId = null` on all pins.
- **Delete hex** (resize) → delete pins with `hexKey` matching the deleted
  hex; delete `state.localMaps[hexKey]`. (Slice 14.11 already pushes a
  pre-resize snapshot — that already captures pins now that the snapshot
  is wider.)
- **Delete pin** is the existing rule; verify the pin editor's delete
  button calls the cascade-aware deletion (clears `pinId` on referencing
  NPCs, rumors, quests; removes pin id from event `pinIds`).
- **Clear hex** (terrain → unknown) → does NOT delete pins or local map
  (existing rule extended).

**Status feedback.** Toasts: "Pin added", "Image stored", "Pin deleted".

**Performance check.** A hex with 20 pins should still render without
lag. The local map canvas with a 1600×1067 image and 20 pins should open
in well under a second.

**Documentation.**
- `data-model.md`: add `LocalMap` shape under map section; expand the
  Pin section with a note about the world-map vs local-map rendering;
  document the new cascade rules; bump the schemaVersion line and the
  v13 → v14 migration entry.
- `ROADMAP.md`: move slice 15 to Done; mention pins now have a UI and
  cross-link UI is in 17/18.

**Edge cases to verify:**
- Local map for a hex that gets deleted by resize: modal closes if open;
  `state.localMaps[hexKey]` removed; pins for that hex removed.
- Faction deletion while a pin in that faction is selected: pin's
  `factionId` becomes null; faction selector reflects "None"; dot
  recolors to type default.
- localStorage near-quota: if `save()` throws (e.g. uploaded image
  pushes total over the limit), catch the error, show a toast
  ("Storage full — image not saved"), revert the in-memory image. Don't
  silently lose data.
- Two pins at exactly the same (x, y): both render, latest-added on top
  due to render order. Drag still picks the topmost.

---

## Out of scope

- Pin ↔ NPC, pin ↔ rumor, pin ↔ quest cross-link UI (slices 17, 18).
- Player view mode (show only discovered pins).
- Multiple local maps per hex (one per hex for v1).
- Local map zoom/pan (canvas at fixed scale within scrollable wrapper).
- Drawing tools on local map (no freehand sketch, no overlay shapes).
- Pin layers, groups, hierarchical pins.
- Connection lines between pins.
- IndexedDB image storage (still localStorage for now).
- Importing existing image collections in bulk.
- Pin search / filter across the whole world.

## Files

- `index.html` — all behavior + CSS for modal and pin editor.
- `data-model.md` — schema v14, LocalMap shape, Pin rendering notes,
  cascade rules.
- `ROADMAP.md` — move slice 15 to Done.

## Definition of done

- All four stages land cleanly with no regressions.
- Pins persist across reload at the v14 schema.
- World map shows pin dots; local map modal renders image + pins.
- Add/drag/edit/delete pins work both from the hex info panel and on the
  local map.
- Undo covers pin and local map operations under the same 50-entry stack
  as 14.11.
- Cascades implemented for faction-delete, hex-delete-via-resize.
- Loading a slice-14.11 save migrates cleanly to v14.
