# Slice 17 — Cross-link Pass 1 (pin ↔ faction, pin ↔ NPCs)

## Goal
Surface the pin-centric cross-references that already exist in the schema. The
Pin entity (slice 15, schema v14) already has `factionId`, and NPCs already
have `pinId` — but neither relationship is editable from the pin side, and the
reverse views (faction → its pins, NPC → its pin) don't show anything. This
slice wires up the UI from the pin's perspective.

## Non-goals
- No schema change. Every target field already exists.
- No NPC-side pin picker — that lands in slice 18.
- No rumor or hex linking — slice 18.
- No quest cross-link polish (jump-buttons / quest reverse views) — slice 18,
  per the roadmap note "cross-link UI pending slice 18" on slice 16.
- No session tagged-list autocomplete — slice 18.

## Schema
No changes. Schema stays at v14. **No localStorage backup needed.**

Field reminders:
- `Pin.factionId: string | null`
- `NPC.pinId: string | null`
- `_cascadeDeletePin(id)` (already in place from slice 15) handles
  `state.npcs`, `state.rumors`, `state.quests`, `state.events.pinIds`.
- All pin edits go through `_pushMapUndo()` — pin faction changes and NPC
  roster changes are pin-adjacent edits and should follow that pattern so the
  map undo stack stays coherent.

## Stages

### Stage 1 — Pin editor faction picker
Add a "Faction" field to the pin editor (in `_renderPinSection` / the local-map
modal pin detail panel — wherever pin name and notes are currently edited).

- Single-select dropdown; mirror the pattern used in `renderQuestDetail` and
  `renderNpcDetail` (faction selector with `<option value="">No faction</option>`
  followed by every faction by name).
- Reuse the inline `iSel` style string already used in those views for visual
  consistency.
- Writes via a new `setPinFaction(id, val)` writer that goes through
  `_pushMapUndo` → `_invalidatePinCache` → `save` → `drawWorld` → `renderMapInfo`,
  matching the existing `setPinField` pattern.
- Show a small color dot (`f.color`) next to the picker once a faction is
  selected. Reuse the existing `.faction-dot` CSS class.

Acceptance:
- Selecting a faction persists across reload.
- Setting it back to "No faction" stores `null`.
- Pin dot on the world map recolours correctly via the existing
  `pinDisplayColor(pin)` (which already prefers `factionId` color).
- Deleting the faction clears the pin's `factionId` (existing cascade in
  the faction-delete handler — verify it actually fires).
- Map undo correctly steps the faction change forward and back.

### Stage 2 — Pin editor NPCs roster
Add an "NPCs at this location" section to the pin editor.

- Lists every NPC with `n.pinId === pin.id`. Each row: disposition pip · name
  · role. Reuse `_NPC_DIP_COLORS` for the pip.
- "+ Add NPC" button reveals a picker (`<select>`) of NPCs whose
  `pinId !== pin.id`. Picking writes that NPC's `pinId` to this pin (wrapped
  in `_pushMapUndo` for undo coherence).
- Each row has a remove button (✕) that sets the NPC's `pinId = null` (also
  through `_pushMapUndo`).
- "+ New NPC here" button creates a new NPC with `pinId` pre-set to this pin
  and routes to the NPCs tab focused on the new NPC. Mirror the existing
  `addNpcToFaction` precedent.

Acceptance:
- Add/remove updates the row immediately and persists.
- The same NPC can never appear twice in the roster.
- Creating a new NPC from the pin lands on the NPCs tab with that NPC
  selected and `pinId` already set.
- Map undo correctly steps roster changes forward and back.

### Stage 3 — Reverse views (read-only)
Surface the back-references on faction and NPC detail.

- **Faction detail**: add a "Locations" section in `renderFactionDetail`,
  parallel structure to the existing "Key NPCs" section (same `.section-box`
  styling, same row layout). List every pin where `pin.factionId === f.id`.
  Each row: pin-type glyph · pin name · hex key. Clicking a row jumps to the
  local map for that pin's hex with the pin selected.
- **NPC detail**: in the Identity section of `renderNpcDetail`, add a "Pin"
  read-only display directly below the Faction picker. Shows pin name + hex
  key, or "—" if `pinId === null`. Small "Open" button next to it jumps to
  the local map for that pin's hex with the pin selected. (The picker
  itself comes in slice 18.)

Acceptance:
- Both reverse views render correctly whether the underlying field is set or
  null.
- "Open" / row-click navigation lands on the right local map with the right
  pin selected.
- A deleted pin (existing cascade) leaves the NPC's pin display showing "—"
  with no broken link.

### Stage 4 — Cross-tab navigation polish
Consistent click-through from pin-related labels:

- Faction name shown anywhere on the pin editor → `goToTab('factions',
  { activeFactionId })`.
- NPC names in the pin's NPC roster → `goToTab('npcs', { activeNpcId })`.

Use the existing `goToTab` and `goToEvent` precedents.

Decide and document: does navigating away close the local-map modal, or leave
it open underneath? Pick one and stick with it.

Acceptance:
- Clicking a faction or NPC label from the pin editor navigates correctly and
  selects the right entity in its tab.
- Modal-close behaviour matches the documented choice.

## Test plan (final manual smoke)
1. Create a faction. Create a pin in some hex. Open the pin → assign the
   faction → reload → verify it persists.
2. Confirm the world-map pin recolours to the faction color.
3. Create two NPCs. From the pin, add both. In the NPCs tab, confirm both
   have `pinId` pointing at the pin.
4. From the pin, remove one NPC → confirm its `pinId` is now null.
5. From the faction detail, confirm the pin appears under "Locations". Click
   → lands on local map with pin selected.
6. From the NPC detail, confirm the pin name shows and Open jumps correctly.
7. Delete the pin (with map undo possible). Confirm the NPC's pin display
   shows "—" and the faction's Locations section no longer lists it. Undo →
   everything restores. Redo → cascade reapplies.
8. Delete the faction. Confirm the pin's Faction picker resets to "No
   faction" and the world-map pin recolours to type-default.

## Out of scope (deferred)
- NPC-side pin picker → slice 18.
- Rumor pin/hex picker → slice 18.
- Quest cross-link polish (jump-buttons, quest reverse views) → slice 18.
- Live count badges on faction sidebar showing pin count — defer.

## Workflow
- Commit `prompts/slice-17.md` first.
- Stage-based: explicit "go" confirmation between stages.
- One commit per stage (or per coherent sub-step within a stage if it gets
  large).
- Update `ROADMAP.md` to move slice 17 from Planned → Done at the end. Update
  the header to "as of slice 17 complete".
- Update `data-model.md` only if anything in the schema actually changes (it
  shouldn't). The Pin section already mentions "Cross-links (UI available
  from slice 17/18 onwards)" — this slice fulfills the 17 half.
