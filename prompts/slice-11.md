# Slice 11: Rumors board with archive

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

Port the Rumor Board from `reference/faction-tracker.html` into a new Rumors tab in the merged app, adapted to the data model.

**Reference:** the Rumor Board UI in `reference/faction-tracker.html` is a grid of cards with filter buttons along the top. That's the general feel we want, with adaptations documented below.

**Out of scope:**
- No cross-linking to pins or hexes beyond what the schema already has (the `pinId` and `hexKey` selectors on rumors will be inert placeholders until slices 15+ and cross-linking slices).
- No events system integration.
- No import from old localStorage keys.

## Schema changes (v6 → v7)

Add to the Rumor entity in `docs/data-model.md`:

```js
archived: boolean,           // default false
createdAt: string,           // ISO timestamp string, e.g. new Date().toISOString()
archivedAt: string | null,   // ISO timestamp when archived, else null
```

**Important:** the Rumor `status` field no longer includes "dangerous". Ensure the migration strips that — if any rumor in existing data has `status: "dangerous"`, migrate it to `status: "unverified"` (and per the earlier design decision, urgency replaces the concept of "dangerous"). This is cleanup from prior planning that may or may not apply — check and handle either way.

Migration v6 → v7:
- For each rumor in `state.rumors`: set `archived = false` if missing, set `createdAt = new Date().toISOString()` if missing (existing rumors get "now" as a fallback), set `archivedAt = null` if missing.
- Fix any `status: "dangerous"` → `"unverified"`.
- Bump `schemaVersion` to 7.
- Update `docs/data-model.md`: Rumor schema block + migration note.

## Layout — Rumors tab

**Tab position:** after Sessions in the main tab bar. Tab label: "Rumors".

**Sub-tabs at the top of the Rumors tab panel:** "Active" (default) and "Archived".
Same styling pattern as the Factions tab's Detail/Relations sub-tabs.

**Active sub-tab layout:**
- Top row: filter buttons (All / Unverified / Confirmed / False / Player Known). "All" is default-selected.
- Below that: a small row of legend/sort info text: "Sorted by urgency (critical first), then newest."
- Main content: a grid of rumor cards using `grid-template-columns: repeat(auto-fill, minmax(270px, 1fr))`.
- Last cell in the grid: a dashed "+ New Rumor" card that creates a new rumor on click.

**Archived sub-tab layout:**
- Same grid, but with an empty-state message if no rumors are archived.
- Archived cards get a subtle visual treatment: reduced opacity (~0.75), a small "archived" badge in the top corner.
- No "+ New Rumor" card on this sub-tab.

**Sub-tab state:** session-only (not persisted). Default to "Active" on every page load.

## Sorting

Within whichever sub-tab is active:

Primary sort: urgency descending (critical → high → medium → low). Map urgency to numeric keys:
```js
const URGENCY_ORDER = { critical: 0, high: 1, medium: 2, low: 3 };
```

Secondary sort: `createdAt` descending (newest first). Use string comparison on ISO strings since they sort correctly lexically.

---

## Stage 1 — Schema migration + tab scaffolding + empty state

**Migration:**
- Implement v6 → v7 exactly as specified above.
- Update `docs/data-model.md`: Rumor entity block (add the three new fields), migration notes, and a brief line about the "Rumors tab" in the architecture section if one exists.
- Bump `schemaVersion` to 7.

**Tab scaffolding:**
- Add a "Rumors" tab button to the main header tab bar, placed after Sessions.
- Add the content panel with two sub-tabs: "Active" and "Archived".
- Both sub-tabs show empty state placeholders: "No rumors yet" for Active, "No archived rumors" for Archived.
- No rumor CRUD yet. No filter row, no grid. Just the shell.

**Styling:** match the existing app — Cinzel labels on sub-tabs, gold accents, same sub-tab pattern as Factions.

Summarize what you changed.

---

## Stage 2 — Rumor CRUD, card grid, filters, sorting

**Rumor creation:**
- Add `newRumor()` function. Creates a rumor with:
  - `id: uid()`
  - `text: ""`
  - `status: "unverified"`
  - `urgency: "medium"`
  - `playerKnown: false`
  - `factionId: null`
  - `npcSourceId: null`
  - `pinId: null`
  - `hexKey: null`
  - `archived: false`
  - `createdAt: new Date().toISOString()`
  - `archivedAt: null`

**Active sub-tab:**
- Filter button row: All, Unverified, Confirmed, False, Player Known. Active filter persisted in `state.ui.rumorFilter` (already in the data model).
- Sort per the spec: urgency desc, then createdAt desc.
- Filter logic:
  - "All" shows all non-archived rumors.
  - "Unverified" / "Confirmed" / "False" filter by `status`.
  - "Player Known" filters by `playerKnown === true`.
  - None of these show archived rumors — archive is strictly separate.

**Rumor card:**
- Top border colored by linked faction (if `factionId` is set), else transparent.
- Status pills row at the top: Unverified / Confirmed / False — click to set status. Selected state visually distinct.
- Urgency picker: small pills or dropdown for Low / Medium / High / Critical. Critical visually distinct (red accent).
- Text area for the rumor itself, placeholder: "Write the rumor as the players might hear it…"
- Bottom row:
  - Faction dropdown (all factions + "No faction link"). Selecting one colors the top border.
  - NPC source dropdown (all NPCs from `state.npcs` with their faction in parens, plus "No NPC source"). Populated from `state.npcs`, not by digging into each faction.
- Footer row: 
  - Player-known pill (toggle button, same style as elsewhere). Label: "⚑ Players Know" when on, "◎ Hidden" when off.
  - Archive button (small icon button, e.g. 📦 or just "Archive"). On click: sets `archived = true`, sets `archivedAt = new Date().toISOString()`, saves, re-renders (card disappears from Active).
  - Delete button (×). Confirm before deleting.
- The `pinId` and `hexKey` fields stay in the schema but get NO UI yet — leave them as null on creation, future slice will add pickers.

**Archive sub-tab:**
- Same grid, shows only archived rumors.
- Each card has the same displays (status, text, urgency, links) but with reduced opacity (~0.75).
- Small "ARCHIVED" badge in the top corner.
- Archive button is replaced with "Unarchive" button — on click: sets `archived = false`, sets `archivedAt = null`, saves, re-renders.
- Delete still available.
- Empty state when no archived rumors.

**Data binding:**
- Generic `updateRumorField(id, field, value)` for all fields.
- `toggleRumorKnown(id)`, `archiveRumor(id)`, `unarchiveRumor(id)`, `deleteRumor(id)`.
- `setRumorStatus(id, status)`, `setRumorUrgency(id, urgency)`.

Summarize.

---

## Stage 3 — Polish + regression checks

**Polish:**
- Confirm filter buttons have consistent visual treatment with other filter-button patterns in the app.
- Sort stability: if two rumors have identical urgency and createdAt (unlikely but possible), fall back to `id` as tertiary sort for deterministic ordering.
- The "+ New Rumor" dashed card should visually match other add-card patterns if any exist; otherwise match the reference `faction-tracker.html`'s "Add Rumor" card style.
- When a rumor's faction is deleted (via Factions tab's delete), rumors referencing it should clear `factionId` to null. This is already in the cascade rules — just verify it's in place for the slice 3 faction-delete handler and the top border color falls back correctly.
- When an NPC is deleted (via the Factions tab's NPC delete), rumors with `npcSourceId` matching should clear that field to null.

**Regression checks — all must work:**
1. Factions tab (Detail + Relations) renders and works.
2. Players tab (sidebar + detail, Tier 1 sheet) works. `currentHexKey` validation still works against the signed coord system from slice 10.
3. Sessions tab CRUD + attendance + layers all functional.
4. Map tab: terrain painting, explored toggle, hex notes, info panel, resize modal all work.
5. All data persists through page reload.
6. The `wm_unified_v1` localStorage blob has `schemaVersion: 7` and every rumor has the three new fields.
7. No console errors on any tab switch.

**Deliverables:**
- Summarize everything I should test across all three stages.
- Flag any cascade edge cases you handled explicitly (faction-delete clearing factionId, NPC-delete clearing npcSourceId).
- Note any UX decisions made that weren't specified in the prompt (e.g. exact button icons, placement of "ARCHIVED" badge, etc.).