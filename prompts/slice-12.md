# Slice 12: NPCs tab

## How to work this file

This slice has three stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation ("go", "yes", "proceed") before starting the next stage.
   If I ask for fixes instead, address those and then ask again.

Do NOT proceed past a stage without explicit confirmation. Do NOT batch stages.
Do NOT rewrite the whole `index.html`. Make surgical edits only.

If a stage's implementation looks like it will exceed ~600 lines of additions,
pause partway through and check in before continuing.

## Committing

Do NOT run git commands. I will handle all commits myself after reviewing each stage.

## Slice context

Add a top-level NPCs tab providing a searchable, filterable view of every NPC across the campaign — whether they belong to a faction or are independent.

**Why this tab exists:** the schema has always supported independent NPCs (factionId: null), but currently the only place to create/edit NPCs is inside the Factions tab. Independent NPCs had no home. This slice fixes that and becomes the primary NPC surface going forward.

**Out of scope:**
- No cross-linking to pins or hexes (the pinId field stays inert until slice 17+).
- No NPC-to-NPC relationships (future slice).
- No NPC character sheet / stats (future slice — NPCs will grow a Tier-1-style sheet eventually, so design the detail view to accommodate that without visual surgery).
- No per-NPC event log UI (Events system is slice 13; integration is later).

## Schema changes (v7 → v8)

Add to the NPC entity in `docs/data-model.md`:

```js
disposition: "friendly" | "neutral" | "hostile" | "unknown",  // default "unknown"
description: string,     // physical description, voice, mannerisms — single freeform textarea
secrets: string,         // GM-only multi-line notes: "what does this NPC know?"
playerKnown: boolean,    // default false — have the players encountered / learned of this NPC?
```

Migration v7 → v8:
- For each npc in `state.npcs`: set `disposition = "unknown"`, `description = ""`, `secrets = ""`, `playerKnown = false` if any are missing.
- Bump `schemaVersion` to 8.
- Update `docs/data-model.md`: NPC entity block with new fields, migration notes.

## Layout — NPCs tab

**Tab position:** after Rumors in the main tab bar. Tab label: "NPCs".

**Overall layout:** sidebar + detail, matching the Factions/Players/Sessions pattern.

**Sidebar structure (top to bottom):**
1. "+ New NPC" button at top. Creates an independent NPC (factionId: null) and selects it.
2. **Filter controls block** (small section with label "Filters"):
   - Search box: text input, filter by name substring (case-insensitive).
   - Faction dropdown: options are "All factions", each faction's name, and "Independent only". Default: "All factions".
   - Player-known filter: single toggle button labeled "Player-known only" — when on, shows only NPCs with `playerKnown === true`. Off by default.
3. **NPC list** below the filter block: shows all NPCs matching the current filters, sorted alphabetically by name. Each entry shows:
   - Faction color dot on the left (grey dot for `factionId === null` / independent)
   - NPC name (fallback "Unnamed")
   - Role in smaller, dimmer text below the name
   - Small disposition indicator (colored pip: green=friendly, grey=neutral, red=hostile, no pip for unknown)
   - Active/selected NPC highlighted as in other sidebars

Selection persisted via `state.ui.activeNpcId`.

**Detail view structure:**
Design this as a sectioned layout that can accommodate future additions. Use section-box pattern throughout.

1. **Identity section:**
   - Name input (large, Cinzel-styled like the faction name input)
   - Role input (freeform — "Captain", "Innkeeper", "Retired Wizard", etc.)
   - Faction dropdown: all factions + "Independent" option. Changing this updates `factionId` (or sets it to null for Independent).
   - Disposition picker: four pills (Friendly / Neutral / Hostile / Unknown), color-coded (green/grey/red/muted), click to select.
   - Player-known toggle: same visual style as faction `playerKnown` toggle. When on, shows "Players know this NPC"; when off, "Hidden from players".
2. **Description section:** a single textarea. Placeholder: "Physical appearance, voice, mannerisms..."
3. **Notes section:** textarea for general GM-side notes (the existing `notes` field). Placeholder: "General notes, backstory, motivations..."
4. **Secrets section:** textarea, visually distinct (dimmer bordered panel, maybe slightly different background to signal "hidden"). Placeholder: "What does this NPC know? (GM-only — never shown to players)"
5. **Delete NPC** button at the bottom, confirm before deleting.

**Empty state** (no NPC selected): centered message, glyph, italic text — consistent with other empty states in the app.

## Factions tab — NPC section simplification

In the Factions tab's Detail sub-tab, the existing "Key NPCs" section changes significantly:

- Remove the full inline NPC editing UI (name/role/notes inputs, add-new-NPC form, NPC log entries).
- Replace with a compact list: one row per NPC belonging to this faction.
- Each row shows: NPC name (fallback "Unnamed"), role (dim text), disposition pip, and a small "Edit →" button.
- Clicking "Edit →" switches to the NPCs tab and selects that NPC.
- An "+ Add NPC to this Faction" button below the list creates a new NPC with `factionId` pre-set to the current faction, switches to the NPCs tab, and selects it for editing.
- No inline editing remains in the faction view.

**Important:** do NOT delete existing NPC data during this simplification. The data model is untouched; only the UI is simplified. Every NPC's `name`, `role`, `notes`, etc. remains in `state.npcs` exactly as before.

## Helper needed

Add a utility function to jump between tabs programmatically:
```js
function goToTab(tabName, contextUpdates) { ... }
```
- Sets the active tab (and sub-tab where relevant).
- Applies any `state.ui` updates passed in (e.g. `{ activeNpcId: "abc" }`).
- Re-renders as needed.

This will get reused in future slices for all cross-tab navigation.

---

## Stage 1 — Schema migration + NPCs tab scaffolding

**Migration:**
- Implement v7 → v8 as specified above.
- Update `docs/data-model.md`: NPC entity with four new fields, migration notes.

**Tab scaffolding:**
- Add "NPCs" tab button to the header tab bar, after Rumors.
- Add content panel: sidebar + detail layout (match existing patterns).
- Sidebar shows just: "+ New NPC" button and empty NPC list (static "No NPCs yet" placeholder if empty, or a simple list with names for now if `state.npcs` has content — no filtering UI yet).
- Detail view shows empty state placeholder.
- Add `state.ui.activeNpcId` (default null).

**Also:** implement the `goToTab(tabName, contextUpdates)` helper, but don't wire it up to anything yet — future stages use it.

No filter controls, no detail sections, no Factions tab changes yet. Just the shell.

Summarize what you changed.

---

## Stage 2 — NPC detail view + full CRUD + filter controls

**Detail view:**
Build all five sections as specified in the "Detail view structure" above: Identity, Description, Notes, Secrets, Delete. 

Data binding:
- Generic `updateNpcField(id, field, value)` for all fields (including the existing name, role, factionId — which, note, was already in the Factions tab but now reachable from here too).
- `setNpcDisposition(id, disposition)`, `toggleNpcKnown(id)` helpers for pills/toggles.
- `deleteNpc(id)` — confirms, then deletes. Follows cascade rules from the data model: clear `npcSourceId` on any rumor referencing this NPC.

**Filter controls in the sidebar:**
- Search box (filters list by name substring, case-insensitive, live on every keystroke).
- Faction dropdown (options: "All factions", each faction, "Independent only"). Default: "All factions".
- "Player-known only" toggle button (default off).
- Filter state persisted in session memory but NOT to localStorage — resets to defaults on page reload.

**NPC list:**
- Render all NPCs matching current filters, sorted alphabetically by name.
- Each entry: faction color dot (or grey), name, role (dim, smaller), disposition pip.
- Click to select.
- Empty-matching-filter state: "No NPCs match these filters" in the list area.
- Empty-no-NPCs-at-all state: "No NPCs yet. Click + New NPC to begin."

**New NPC creation:**
- `+ New NPC` button: creates an NPC with `{factionId: null, name: "", role: "", notes: "", disposition: "unknown", description: "", secrets: "", playerKnown: false, pinId: null}`, adds to `state.npcs`, selects it.

Summarize.

---

## Stage 3 — Factions tab NPC section simplification + cascade + regression checks

**Factions tab — Detail sub-tab — "Key NPCs" section:**
- Remove the full inline editing UI.
- Replace with the simplified list (name, role, disposition pip, "Edit →" button per row).
- Below the list: "+ Add NPC to this Faction" button.
- Clicking "Edit →" on a row: use `goToTab("npcs", {activeNpcId: npc.id})` to switch.
- Clicking "+ Add NPC to this Faction": create an NPC with `factionId` pre-set to the current faction, then `goToTab("npcs", {activeNpcId: newNpc.id})`.

**Verify existing cascade rules:**
- When a faction is deleted (existing Factions tab behavior): all NPCs with that `factionId` get `factionId = null` (they become independent, not deleted). This was already in slice 3's cascade — just verify it still works and that the NPCs tab reflects the change.
- When an NPC is deleted (from NPCs tab): rumors with `npcSourceId === this.id` get `npcSourceId = null`. This is the new cascade path introduced by the NPCs tab having a delete button.

**Regression checks — all must work:**
1. Factions tab Detail view loads, shows the simplified NPC list. "+ Add NPC" flow correctly jumps to NPCs tab with the new NPC selected and `factionId` pre-populated.
2. Relations sub-tab works.
3. Players tab (sidebar + Tier 1 sheet) works.
4. Sessions tab (CRUD, attendance, layers, tagged lists) works.
5. Rumors tab (Active + Archived sub-tabs, filters, NPC source dropdown) works — and the NPC source dropdown should still correctly list NPCs from the flat `state.npcs` array.
6. Map tab (painting, explored, notes, resize) works.
7. `schemaVersion` is 8 in localStorage.
8. Every NPC in localStorage has the four new fields.
9. No console errors on any tab switch.
10. Delete an NPC from the NPCs tab; verify any rumor that had it as source now shows "No NPC source" in the dropdown.

**Deliverables:**
- Summarize everything I should test.
- Note any UX decisions not in the spec.
- Flag whether the `goToTab` helper is implemented cleanly enough to reuse in future cross-linking slices, or if it needs refactoring.