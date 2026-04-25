# Slice 13: Events system

## How to work this file

This slice has four stages. For each stage in order:

1. Read the stage section below.
2. Implement exactly what it describes — no more, no less.
3. When finished, STOP and report:
   - A summary of what you changed.
   - What I should test before proceeding.
   - "Ready for Stage N+1?" as the final line.
4. Wait for my explicit confirmation before starting the next stage.
   Confirmations will be short ("go", "yes", "proceed", "continue").
   If I ask for fixes instead, address those and then ask again.

Do NOT proceed past a stage without explicit confirmation. Do NOT batch stages.
Do NOT rewrite `index.html` wholesale — make surgical edits only. Keep changes
scoped to the sections you need to touch.

If a stage looks like it will exceed ~600 lines of additions, pause partway
through and check in before continuing.

## Slice context

Add an Events tab — a general log of things that happen **outside** sessions
(a faction makes a move between sessions, a rumor proves false, an NPC dies
off-screen, etc.). Events already exist in the data model (schema v8); their
cascades are already wired into `deleteFaction`, `deleteNpc`, and
`deleteSession`. What's missing is the UI.

Events also gain two new schema fields in this slice — a "time until" clock
and a "duration" clock — so the slice bumps the schema from v8 to v9.
**Back up `wm_unified_v1` from localStorage before starting Stage 2.**
Stage 1 is v8-compatible and safe; Stage 2 introduces the migration.

### In scope

- New top-level **Events** tab, slotted in the nav between **Sessions** and
  **Rumors**.
- Two-pane layout modeled on `#panel-npcs` from slice 12: sidebar list on the
  left with a filter bar, detail pane on the right.
- CRUD on `state.events`.
- **Two clock widgets per event**, mirroring the faction clock UX:
  - `timeUntilClock` — countdown to when the event happens / triggers.
  - `durationClock` — how long the event lasts once it starts.
  - Each is `{ size: 4|6|8|10|12, filled: number } | null`. Null means
    the slot is empty and shows a `+ Add` button. Unlike faction clocks,
    these have no `id`, no `label` (the slot is the label), and no
    `consequence` (the event itself is the consequence).
- Link editing UI for all six link fields on `Event`:
  - `factionIds: [string]`    — multi-select chips
  - `npcIds:     [string]`    — multi-select chips
  - `pinIds:     [string]`    — multi-select chips (list may be empty today;
    slice 15 populates pins — UI still renders correctly with zero pins)
  - `hexKeys:    [string]`    — multi-select chips (picker lists existing
    `state.hexes` keys; no click-to-pick-on-map this slice)
  - `questId:    string|null` — single-select dropdown
  - `sessionId:  string|null` — single-select dropdown
- Filters: full-text (`text` + `date`), faction, urgency, player-known-only.
- New / delete / player-known toggle / urgency pills.
- Empty states for both the list (no events) and the detail pane (nothing
  selected).
- `docs/data-model.md` updates (stage 4 only).

### Out of scope — do not build in this slice

- Contextual "+ Event" buttons on faction / NPC / quest / session cards.
  (Slice 17/18 — cross-link passes.)
- Mirrored "recent events" sections inside other entities' cards.
  (Slice 17/18.)
- Click-to-pick a hex on the map to add a `hexKey` link.
  (Later polish.)
- Autocomplete / suggestions derived from session tagged lists.
  (Not planned.)
- Timestamps (`createdAt`, `updatedAt`) on events. Explicitly out of scope
  per `data-model.md`.
- Any change to the `Event` schema beyond the two clock fields named above.
- User-labeled / arbitrary-count clocks like factions have. Events get
  exactly two semantic slots; that's the whole feature.
- Toast notifications when a clock fills (faction clocks toast; events
  don't, this slice).

### State additions

In `state.ui` (initial shape around line 1079 of `index.html`):

```js
activeEventId:          null,
eventFilter:            '',
eventFactionFilter:     '',    // '' = all factions
eventUrgencyFilter:     '',    // '' = all urgencies
eventPlayerKnownOnly:   false,
```

In `load()`, defensively default these fields if absent from the parsed state
(same pattern used for the slice 11/12 additions). No version bump, no
`migrateToV9()` — this is a v8 UI-state additive change only.

### Naming / CSS conventions

- CSS class prefix for event-specific styles: `e-*` (e.g. `.e-list-row`,
  `.e-filter-bar`, `.e-chip`, `.e-chip-picker`).
- Reuse the shared palette variables (`--gold`, `--gold-dim`, `--text-dim`,
  `--panel-mid`, `--border`, etc.) — do not introduce new color values.
- Reuse `.sidebar`, `.sidebar-header`, `.sidebar-list`, `.f-content`,
  `.empty-state`, `.btn-small`, `.btn-danger`, `.hidden`, `.threat-pill`
  where they fit. If you need a new style, add it next to the existing
  event styles in a clearly-labeled block.
- Mirror the `panel-npcs` DOM shape: `.sidebar` with `.sidebar-header` +
  filter controls + `.sidebar-list`, then `.f-content` with an
  `.empty-state` element and a hidden detail container.

### Tab placement

Update the tab nav (around line 819) to insert the Events tab between
Sessions and Rumors:

```html
<button class="tab-btn" data-tab="map">Map</button>
<button class="tab-btn" data-tab="factions">Factions</button>
<button class="tab-btn" data-tab="quests">Quests</button>
<button class="tab-btn" data-tab="sessions">Sessions</button>
<button class="tab-btn" data-tab="events">Events</button>   <!-- NEW -->
<button class="tab-btn" data-tab="rumors">Rumors</button>
<button class="tab-btn" data-tab="npcs">NPCs</button>
<button class="tab-btn" data-tab="players">Players</button>
```

And add a matching `<div class="panel" id="panel-events">…</div>` in the
panels section, placed next to `panel-sessions` or `panel-rumors`.

---

## Stage 1 — Tab shell, CRUD, filter bar

Goal: you can click the Events tab, create an event, edit its basic fields,
filter the list, delete it, refresh the page, and it all persists. No link
editing yet.

### What to add

1. **Tab button + panel** per the "Tab placement" note above.
2. **State additions** per the "State additions" note above, plus defensive
   defaults in `load()`.
3. **Panel markup** in `#panel-events`:
   - `.sidebar`
     - `.sidebar-header`: title "Events" + `+ New Event` button that calls
       `newEvent()`.
     - `.e-filter-bar` (a small vertical stack or flex-wrap row mirroring
       `.npc-filters`):
       - text input, placeholder "Search text or date…"
       - `<select>` populated with all factions (value = faction id;
         default option "All factions").
       - `<select>` with options All urgencies / Low / Medium / High / Critical.
       - Toggle button "Player-known only" (active/inactive visual, mirror
         `.npc-known-btn`).
     - `.sidebar-list` with `id="event-list"`.
   - `.f-content`
     - `.empty-state` with `id="event-empty"` and the standard glyph +
       "Select an event or create one to begin." message.
     - `<div id="event-detail" class="hidden"></div>` — the detail pane.

4. **Functions** (group them together in the source under a
   `// ── Events ──` banner comment, following the existing file's
   section-heading convention):

   - `renderEventsPanel()` — top-level re-render. Populates the faction
     `<select>`, calls `renderEventList()` and `renderEventDetail()`.
   - `renderEventList()` — applies filters to `state.events`, renders each
     matching event as a `.e-list-row`:
     - left-aligned: `date` (or "—" if empty), first ~60 chars of `text`
       (or "(no description)" if empty).
     - right-aligned metadata: urgency pip (color-coded), faction color
       dots (up to 3), small count badges for NPCs/hexes/pins if > 0, a
       locked glyph `🔒` (or existing convention) if `!playerKnown`.
     - highlighted when `id === state.ui.activeEventId`.
     - click → `selectEvent(id)`.
   - `renderEventDetail()` — renders `#event-detail` for the active event:
     - top row: date input (freeform), urgency pills, player-known
       toggle, Delete button (calls `deleteEvent(id)` with confirm).
     - Urgency widget: a 4-pill row matching the rumors pattern. Add a
       new `.e-urgency-row` + `.e-urgency-pill` CSS block mirroring
       `.rumor-urgency-row` / `.rumor-urgency-pill` (lines ~493–503 of
       `index.html`) — same border / color / active-state behavior for
       low/medium/high/critical. Don't reuse `.rumor-urgency-pill`
       directly; create the parallel class so events can evolve
       independently.
     - text `<textarea>` (large, primary content).
     - Leave a clearly-marked empty `<div class="e-links"></div>` section
       below the textarea as a placeholder for Stage 3 — do NOT build link
       UI yet.
   - `newEvent()` — prepends a new event to `state.events` with:
     ```js
     { id: uid(), date: '', text: '', urgency: 'low', playerKnown: false,
       factionIds: [], npcIds: [], pinIds: [], hexKeys: [],
       questId: null, sessionId: null }
     ```
     sets `state.ui.activeEventId` to the new id, saves, re-renders.
   - `selectEvent(id)` — sets `activeEventId`, saves, re-renders.
   - `deleteEvent(id)` — `confirm()` prompt; on yes, splices out of
     `state.events`; if `activeEventId === id`, clears it; saves; re-renders.
   - Field-update handlers that mutate the active event and call
     `save()` + `renderEventList()` on change. The text and date inputs
     should update without re-rendering the whole detail pane (so the
     cursor doesn't jump) — mirror the pattern used on NPCs / sessions.

5. **Tab switch wiring** — the generic tab-switch logic at line ~1235
   already handles any tab; confirm `renderEventsPanel()` is called when
   the tab becomes active (add a case if the code dispatches per tab).

### Constraints

- No link-editing UI yet. Leave the `.e-links` container empty.
- Faction `<select>` in the filter bar must rebuild whenever factions
  change (or on every `renderEventsPanel()`), so newly-created factions
  show up without a reload.
- Do not touch `deleteFaction` / `deleteNpc` / `deleteSession`. Their
  event cascades are already correct.

### Test checklist for Stage 1

- Click Events tab. Empty state renders.
- `+ New Event` creates a row, selects it, shows the detail pane.
- Type a date, type some text, change urgency, toggle player-known. All
  persist after refresh.
- Create three events. Filter by text — matches on both `text` and `date`
  substrings (case-insensitive).
- Create two factions (if you don't already have them). Faction filter
  narrows to events whose `factionIds` includes the chosen faction — but
  since links aren't editable yet, this will match nothing for new events.
  That's expected. Confirm the dropdown populates correctly.
- Urgency filter narrows correctly.
- Player-known-only toggle hides unknown events.
- Delete an event — confirms, removes, empty state returns.
- `state.ui.activeEventId`, `eventFilter`, `eventFactionFilter`,
  `eventUrgencyFilter`, `eventPlayerKnownOnly` all persist in localStorage.

---

## Stage 2 — Clocks (schema v9)

Goal: each event has two optional clock widgets — "Time Until" and
"Duration" — visually mirroring the faction clock UX. Schema bumps to v9.

**Before starting this stage:** back up `wm_unified_v1` from localStorage.
F12 → Application → Local Storage → copy the value to a file.

### Schema change

Add two fields to the `Event` entity:

```js
{
  ...existing event fields,
  timeUntilClock: { size: 4|6|8|10|12, filled: number } | null,
  durationClock:  { size: 4|6|8|10|12, filled: number } | null,
}
```

Defaults: both null. `filled` is clamped to `[0, size]`. Default size on
add: 6 (matches faction default).

Note the deliberate simplification vs. faction clocks: no `id` (the slot
identifies it), no `label` (the slot IS the label), no `consequence`
(the event itself is the consequence). Just size and fill.

### What to add

1. **Bump schema version**:
   - Change `schemaVersion: 8` to `schemaVersion: 9` wherever it's
     written (state init, `save()` if it explicitly sets it).
   - In `load()`, add `if (v < 9) migrateToV9(parsed);` to the migration
     chain. Update the version-guard line (`if (v <= 8)` → `if (v <= 9)`)
     so the loader still accepts v9 saves.

2. **Migration function**:
   ```js
   function migrateToV9(parsed) {
     parsed.events = parsed.events || [];
     parsed.events.forEach(e => {
       if (!('timeUntilClock' in e)) e.timeUntilClock = null;
       if (!('durationClock'  in e)) e.durationClock  = null;
     });
   }
   ```
   Place it next to the existing migration functions, in numeric order.

3. **Update `newEvent()`** (from Stage 1) to initialize the two clock
   fields as null.

4. **Generalize `renderClockSVG`** (currently at ~line 2113):
   - Current signature: `renderClockSVG(clock, color, clockId)`. The
     onclick is hardcoded to `stepClockSeg('${clockId}',${i})`.
   - New signature: `renderClockSVG(clock, color, onSegClickAttr)` where
     `onSegClickAttr` is a string template with `{i}` replaced per
     segment. Example callers:
     - Faction:  `renderClockSVG(clock, f.color, "stepClockSeg('"+clock.id+"',{i})")`
     - Event "time until":  `renderClockSVG(clk, '#c8a030', "stepEventClockSeg('"+e.id+"','timeUntil',{i})")`
     - Event "duration":    `renderClockSVG(clk, '#a09070', "stepEventClockSeg('"+e.id+"','duration',{i})")`
   - Update the existing faction call sites accordingly so factions
     continue to work unchanged.
   - **Fallback**: if generalizing turns out to require touching too
     many call sites or behavior subtly differs, instead duplicate the
     function as `renderEventClockSVG(clock, color, eventId, slot)`
     with hardcoded event-specific onclick. Note in the stage summary
     which path you took.

5. **Event clock handlers** (new functions, near the existing
   `stepClockSeg` etc.):
   ```js
   function stepEventClockSeg(eventId, slot, segIdx) { ... }
   function setEventClockSize(eventId, slot, size, btn) { ... }
   function addEventClock(eventId, slot)    // null → { size: 6, filled: 0 }
   function removeEventClock(eventId, slot) // → null
   ```
   Where `slot` is `'timeUntil'` or `'duration'` and maps to
   `event.timeUntilClock` / `event.durationClock`. Each handler calls
   `save()` and re-renders only the affected slot (mirror the
   `updateClockDisplay` pattern — don't re-render the whole detail
   pane).

6. **CSS** — add a parallel `.e-clocks-*` block near the existing
   `.clock-*` styles (around line 294). Do not reuse `.clock-section` /
   `.clock-card` directly; create siblings:
   - `.e-clocks-row` — flex row, two slots side by side, gap, padded.
   - `.e-clock-slot` — a single slot. When empty, shows a `+ Add` button
     centered. When filled, shows: small label header ("Time Until" or
     "Duration") above the SVG, size selector below, ✕ remove button in
     the corner.
   - `.e-clock-slot-label` — Cinzel-styled tiny caps header.
   - `.e-clock-add-btn` — the `+ Add` placeholder button styling.
   - Reuse `.clock-size-btn` and `.clock-size-row` (faction styles)
     verbatim — they're generic enough.

7. **Detail-pane integration** — in `renderEventDetail()` (from Stage 1),
   insert the clocks row **between the text textarea and the empty
   `.e-links` placeholder**:
   ```
   [date | urgency | player-known | delete]
   [text textarea]
   [Time Until slot] [Duration slot]   ← NEW
   [.e-links placeholder]              ← stays empty until Stage 3
   ```

8. **List-row indicator** — in `renderEventList()` (from Stage 1), if
   `event.timeUntilClock || event.durationClock`, append a small ⏰
   glyph to the row preview. Tooltip text: e.g. `"Time until: 2/6 ·
   Duration: 0/4"` (omit the side that's null). Place the glyph after
   urgency pip, before the future link badges.

### Constraints

- The two slots are FIXED. Do not add UI for adding a third clock or
  for relabeling. Slot is the label.
- Clock fill behavior matches factions: clicking a segment fills up to
  and including that segment, or clicks-to-uncheck if you click an
  already-filled segment (the existing `stepClockSeg` logic). Reuse
  exactly.
- Removing a clock (✕ button) sets the field back to null with no
  confirmation prompt — clocks are cheap, re-adding is one click.
- No toast on full. Faction clocks toast; events do not (out-of-scope
  per slice context).

### Test checklist for Stage 2

- Reload after deploying Stage 2. Existing events from Stage 1 load
  cleanly — the two new fields default to null via migration.
- `localStorage.getItem('wm_unified_v1')` → parsed JSON shows
  `schemaVersion: 9` and each event has `timeUntilClock: null,
  durationClock: null`.
- Empty event detail pane shows two slots, each with `+ Add` buttons
  and labels ("Time Until" / "Duration").
- Click `+ Add` on Time Until — clock appears at default size 6, fill 0.
  Click segments to fill. Refresh — fill persists.
- Change size 6 → 8 → 4. Filled count clamps to new size when shrinking.
- Click ✕ on a clock — slot returns to `+ Add` state immediately.
- Add both clocks on one event. List-row preview shows the ⏰ glyph
  with tooltip showing both fractions.
- Add only one clock — tooltip shows only that side.
- Faction clocks still work exactly as before. Open an existing
  faction with clocks, click segments, change sizes, add/remove
  clocks. No regressions.
- Schema downgrade safety: if you reload the v8 backup you saved at
  the start of this stage, the migration runs cleanly and brings the
  state to v9. (Manual test: paste backup into localStorage, reload.)

---

## Stage 3 — Entity links (chips + pickers)

Goal: each event's six link fields can be edited via chips and pickers.
Link counts appear in the list row preview.

### What to add

1. **Links section** inside `#event-detail`, replacing the empty
   `.e-links` placeholder from Stage 1. Render six rows, in this order:

   | Row       | Field        | Source                       | UI                      |
   |-----------|--------------|------------------------------|-------------------------|
   | Factions  | factionIds   | state.factions               | chips + add-picker      |
   | NPCs      | npcIds       | state.npcs                   | chips + add-picker      |
   | Pins      | pinIds       | state.pins                   | chips + add-picker      |
   | Hexes     | hexKeys      | `Object.keys(state.hexes)`   | chips + add-picker      |
   | Quest     | questId      | state.quests                 | single-select `<select>`|
   | Session   | sessionId    | state.sessions               | single-select `<select>`|

2. **Chip rendering** (`.e-chip`):
   - Label = the entity's display name:
     - faction → `faction.name`
     - NPC → `npc.name`
     - pin → `pin.name`
     - hex → `"col,row"` literal, optionally suffixed with `hex.name` if set
   - If the referenced entity is missing (dangling id), render chip with
     text `(deleted)` in a muted color. Don't throw.
   - Small × button on the chip removes that id from the array, saves,
     re-renders detail + list.
   - Faction chips take their color from `faction.color` as a left border
     accent, matching the pattern used elsewhere.

3. **Add-picker** (`.e-chip-picker`):
   - Triggered by a `+ Add` button at the end of each chip row.
   - When clicked, replaces the button inline with a small `<select>` (or
     a searchable dropdown if you have one already — `<select>` is fine).
   - The `<select>` lists only entities **not already linked**.
   - Choosing an option appends the id to the array, saves, re-renders
     the row (picker collapses back to `+ Add`).
   - If there are no eligible entities to add (empty source, or all
     already linked), the `+ Add` button is disabled with tooltip like
     "No pins to link" / "All factions linked".

4. **Quest / Session dropdowns**:
   - Plain `<select>` with a `— none —` option + all quests / sessions.
   - On change, set `event.questId` / `event.sessionId`, save, re-render
     list.

5. **List row preview update** (`renderEventList`):
   - After the urgency pip, render small text-only badges for non-zero
     link counts. Example: `2F · 1N · 3H`. Omit zero-count badges. If
     `questId` is set, add a small `Q` glyph; if `sessionId` is set,
     add an `S` glyph with the session's title as a tooltip.

6. **Faction filter behavior** (already built in Stage 1): now becomes
   actually useful — events with a matching `factionIds` entry surface.
   No code change needed, just verify.

### Constraints

- Pickers use the inline-replace pattern, not modals. Events should feel
  as cheap to edit as NPCs.
- Do not add an "events referencing this faction" section inside
  `#panel-factions` — that's slice 17/18.
- Do not add a map-pick button for hexes — that's later polish.

### Test checklist for Stage 3

- Link 2 factions, 1 NPC, 1 hex, 1 pin (if you have one) to an event.
  Refresh. All chips return.
- Remove a faction chip. Count in list row preview decrements.
- Delete a linked faction from the Factions tab. Event's chip for that
  faction disappears automatically (existing cascade). The event itself
  is unaffected.
- Delete a linked NPC. Same result via `deleteNpc` cascade.
- Delete a linked session (from Sessions tab). Event's session dropdown
  resets to "— none —" (existing cascade).
- Link an event to a quest, refresh, verify the dropdown retains the
  selection.
- Faction filter in the list narrows to events linked to that faction.
- List row preview shows `2F · 1N · 1H` badges where appropriate.
- Dangling-id robustness: open devtools, manually push a fake id into
  `state.events[0].factionIds`, call `save()` then reload. Chip renders
  `(deleted)`, not a crash.

---

## Stage 4 — Polish, sort, cascade verification, data-model doc

Goal: the slice lands clean. Documentation reflects reality. No new
features.

### What to do

1. **Sort order**:
   - Events render in array order (newest at top, since `newEvent()`
     prepends). No sort function needed; document this with a brief
     comment in `renderEventList()`.
   - Do NOT sort by the `date` string — it's freeform ("Session 12",
     "1/15", etc.), so alphabetic sort is worse than insertion order.

2. **Cascade verification** — read through the existing delete handlers
   and confirm the following (no code changes expected; note any gaps
   in your stage-end summary):

   | Handler          | Event cascade present?                    |
   |------------------|-------------------------------------------|
   | `deleteFaction`  | YES — filters `factionIds`                |
   | `deleteNpc`      | YES — filters `npcIds`                    |
   | `deleteSession`  | YES — nulls `sessionId`                   |
   | `deletePlayer`   | NOT NEEDED — events don't reference players |
   | `deleteRumor`    | NOT NEEDED — events don't reference rumors  |
   | `deleteQuest`    | N/A — Quests tab is a stub; slice 16 wires it up and MUST cascade `questId` |
   | `deletePin`      | N/A — Pins not yet createable; slice 15 wires it up and MUST cascade `pinIds` |

   Do not add the quest/pin cascades prematurely — but add a one-line
   `// TODO(slice 15/16): cascade to events.pinIds / events.questId on delete`
   comment near where those future handlers will live, if a natural spot
   exists.

3. **Hex clear does NOT remove hexKey from events** — the data-model
   says hexes remain referenced even when terrain is cleared. Verify the
   map "clear hex" / "set terrain to unknown" flow doesn't touch
   `state.events`. It shouldn't; if it does, remove that logic.

4. **Dangling-id chip rendering**: confirm Stage 3's `(deleted)` fallback
   renders for all four multi-id link types. For the single-ref fields
   (`questId`, `sessionId`), take the cleaner approach used elsewhere in
   the codebase — null them out defensively in `load()` if the id
   doesn't resolve to an existing entity. Do this in the same defensive
   block that defaults the new `state.ui` event fields. After this, the
   quest/session dropdowns will naturally show `— none —` for orphaned
   refs without any chip-rendering special case.

5. **`docs/data-model.md` updates**:

   - **Schema version**: bump every reference to v8 in the doc to v9.
     This includes the `schemaVersion: 8` line in the top-level state
     shape, and any prose referencing "schema v8" / "as of v8".
   - **Event entity**: add the two new fields to the `Event` block:
     ```js
     timeUntilClock: { size: 4|6|8|10|12, filled: number } | null,
     durationClock:  { size: 4|6|8|10|12, filled: number } | null,
     ```
     Add a one-paragraph note below the block explaining: each clock
     is optional (null = empty slot); the slot identifies the clock
     (no `id`/`label`/`consequence` fields like faction clocks have);
     `timeUntilClock` represents countdown to the event triggering;
     `durationClock` represents how long the event lasts once it
     starts.
   - **Migration note**: add a brief migration note near the existing
     v1 migration note: "Migration v8 → v9: `migrateToV9()` adds
     `timeUntilClock: null` and `durationClock: null` to every event."
   - `state.ui.activeTab` enum — add `"events"` and `"npcs"`:
     ```js
     activeTab: "map" | "factions" | "rumors" | "quests"
              | "sessions" | "players" | "npcs" | "events",
     ```
   - `state.ui` shape — add the five new event fields alongside existing
     filter fields:
     ```js
     activeEventId:        string | null,
     activeNpcId:          string | null,   // (was missing; add it)
     eventFilter:          string,
     eventFactionFilter:   string,
     eventUrgencyFilter:   string,
     eventPlayerKnownOnly: boolean,
     ```
   - Under "Derived views", add:
     ```
     - Faction → events:  state.events.filter(e => e.factionIds.includes(f.id))
     - NPC → events:      state.events.filter(e => e.npcIds.includes(n.id))
     - Pin → events:      state.events.filter(e => e.pinIds.includes(p.id))
     - Hex → events:      state.events.filter(e => e.hexKeys.includes(key))
     - Quest → events:    state.events.filter(e => e.questId === q.id)
     - Session → events:  state.events.filter(e => e.sessionId === s.id)
     ```
     (Several are already there — merge, don't duplicate.)

6. **ROADMAP.md**: move "Slice 13: Events system" from Planned to Done,
   and update the header "as of slice 12 complete" → "as of slice 13
   complete".

### Test checklist for Stage 4

- Fresh reload: state loads cleanly, no console errors, defaults applied
  for any missing ui fields.
- All six link types render, edit, persist, cascade (where applicable).
- Both clock slots render, edit, persist; faction clocks unchanged.
- `data-model.md` diff: schema version bumped to 9 throughout, Event
  entity gains the two clock fields with note, migration note added,
  enum updated, ui shape updated, derived views section has all six
  new lines (no duplicates).
- `ROADMAP.md` diff: slice 13 moved to Done, header updated.
- Manual grep: no stray `TODO` or `FIXME` left behind from stage 1/2/3
  scaffolding except the two intentional pin/quest cascade TODOs.
- All seven other tabs still work (map, factions, quests, sessions,
  rumors, npcs, players). Nothing regressed.
- localStorage shows `schemaVersion: 9`. A v8 backup, if loaded, still
  migrates cleanly (do not re-test if you already did this in Stage 2).

### Completion summary to write at the end

Report:

- Total lines added to `index.html` (roughly).
- Any surprises or decisions you made.
- Confirmation that slice 13 is complete.
