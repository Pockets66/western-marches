# Slice 16 — Quests tab

Surfaces the Quest entity in the UI. The schema and `state.quests` array
have existed since slice 8, and Events already supports linking to quests
via `questId`. The Quests tab in the navigation is currently a placeholder.

**No schema change.** Schema stays at v14 (slice 15). All four cascade
rules involving quests are already documented in `data-model.md` — this
slice verifies they're implemented and adds the missing ones.

## Motivation

Western Marches campaigns accumulate quest hooks: bounties, errands,
rescue missions, mysteries, faction agendas. Without a UI surface for
them they get scattered across session notes, hex notes, NPC notes, and
rumors — duplicated, contradicted, lost. The Quest entity exists to
collect them in one place with a stable status lifecycle and links to
the rest of the world (giver NPC, sponsoring faction, location, event
history).

This slice also serves as the connective tissue for Slice 18, which adds
quest cross-link UI to NPCs, rumors, and pins.

## Design decisions

- **Sidebar + detail pane layout.** Same shape as factions, sessions,
  NPCs, events. Card-grid (rumors) doesn't suit the higher density of
  quest fields.
- **Active / Archive sub-tabs.** Like factions has Detail / Relations.
  - **Active sub-tab** shows quests with status in `['available',
    'active']`.
  - **Archive sub-tab** shows quests with status in `['completed',
    'failed', 'abandoned']`.
  - Status drives archive membership — no separate `archived` field.
    Setting a quest's status to `completed` (or failed/abandoned) moves
    it to archive. Setting it back to `active` (or available) restores
    it.
  - Sub-tab choice persists in `state.ui.questsSubtab`.
- **Status filter pills inside each sub-tab.** Active sub-tab pills:
  All / Available / Active. Archive sub-tab pills: All / Completed /
  Failed / Abandoned.
- **Sort within a sub-tab.** Urgency (critical first), then title
  alphabetic. No user-configurable sort in v1.
- **Search and player-known toggle apply to the current sub-tab only.**
- **Pin OR hex mutex.** Same rule as rumors. UI enforces: setting
  `pinId` clears `hexKey` and vice versa.
- **Event history readout.** Detail pane shows events filtered by
  `e.questId === q.id` as a read-only list. Click an event row to switch
  to the Events tab with that event selected.
- **Selected quest persists across sub-tab switch.** If a quest selected
  on Active is then marked completed, it stays selected; the user is
  switched to the Archive sub-tab so they don't lose context. Same in
  reverse.
- **No undo extension.** Slice 14.11's undo stack covers map-canvas
  interactions. Quests are form inputs — native textarea undo suffices.
- **No schema delta.** Quest shape stays as documented. Migration is a
  no-op (`state.quests = state.quests || []` defensive only).

## Stages

Each stage is one logical chunk. Stop after each for "go" confirmation.

---

### Stage 1 — Shell, sub-tabs, sidebar, CRUD

End of stage: tab renders with Active / Archive sub-tabs; you can create
/ select / delete quests; sidebar filters by sub-tab + status pill +
search + player-known toggle; detail pane shows title, status, urgency,
and player-known toggle.

**HTML shell.** Replace the `<div class="panel" id="panel-quests">[Quests]
</div>` placeholder with sub-tab bar + sidebar + content layout (sub-tab
shape modeled on `panel-factions`):

```html
<div class="panel" id="panel-quests">
  <div class="subtab-bar">
    <button class="subtab-btn active" data-subtab="active"
      onclick="switchQuestsSubtab('active')">Active</button>
    <button class="subtab-btn" data-subtab="archive"
      onclick="switchQuestsSubtab('archive')">Archive</button>
  </div>
  <div class="quests-pane" id="quests-pane">
    <div class="sidebar">
      <div class="sidebar-header">
        <span>Quests</span>
        <div style="display:flex;gap:8px;align-items:center">
          <span id="q-count" style="color:var(--text-muted)">0</span>
          <button class="btn-small" onclick="newQuest()">+ New</button>
        </div>
      </div>
      <div class="q-filter-bar">
        <div class="q-status-pills" id="q-status-pills"></div>
        <input class="q-search" type="text" id="quest-search-inp"
          placeholder="Search title, summary, notes…"
          oninput="_questSearchChange(this.value)">
        <button class="q-known-btn" id="quest-known-btn"
          onclick="_questToggleKnown()">Player-known only</button>
      </div>
      <div class="sidebar-list" id="quest-list"></div>
    </div>
    <div class="f-content">
      <div class="empty-state" id="quest-empty">
        <div class="glyph">✦</div>
        <p>Select a quest or create one to begin.</p>
      </div>
      <div id="quest-detail" class="hidden"></div>
    </div>
  </div>
</div>
```

**State and constants.**

- Add to `state.ui`:
  - `questsSubtab: 'active' | 'archive'` (default `'active'`)
  - `questStatusFilter: string` (default `''` = all). Distinct values
    per sub-tab; reset to `''` when sub-tab changes.
  - `questFilter` (search string, default `''`)
  - `questPlayerKnownOnly` (boolean, default `false`)
  - `activeQuestId` already exists in the documented top-level UI shape.
- Defensive init for any of these missing on load.
- `Q_STATUSES`: ordered constants
  ```js
  const Q_STATUSES = [
    { key: 'available', label: 'Available', color: 'var(--gold-pale)',
      group: 'active' },
    { key: 'active',    label: 'Active',    color: 'var(--gold)',
      group: 'active' },
    { key: 'completed', label: 'Completed', color: '#7cbc80',
      group: 'archive' },
    { key: 'failed',    label: 'Failed',    color: '#e06060',
      group: 'archive' },
    { key: 'abandoned', label: 'Abandoned', color: 'var(--text-dim)',
      group: 'archive' },
  ];
  ```
- Reuse `URGENCY_ORDER` and the standard urgency colors from rumors/
  events.

**Sub-tab logic.**

- `switchQuestsSubtab(sub)` → set `state.ui.questsSubtab = sub`,
  reset `questStatusFilter` to `''`, save, update `.subtab-btn.active`
  classes, re-render status pills + list. If `activeQuestId` is set and
  that quest's status doesn't match the new sub-tab, leave the detail
  pane as-is (the user can still see/edit the quest; it just won't be
  highlighted in the sidebar list).
- Status pills are computed from `Q_STATUSES.filter(s => s.group ===
  questsSubtab)` plus an "All" pill at the front.

**CRUD functions.**

- `newQuest()` → push a new quest with all defaults (title `''`, status
  `'available'`, urgency `'medium'`, `playerKnown: false`,
  `giverNpcId: null`, `factionId: null`, `pinId: null`, `hexKey: null`,
  `summary: ''`, `reward: ''`, `notes: ''`). New quests always default
  to `available` so they appear on the Active sub-tab. If the user is
  currently on the Archive sub-tab when they hit + New, switch them to
  Active automatically (otherwise the new quest seems to vanish). Set
  `state.ui.activeQuestId = q.id`. Save. Re-render.
- `deleteQuest(id)` → confirm prompt, run cascade (`state.events.forEach
  (e => { if (e.questId === id) e.questId = null; })`), splice from
  `state.quests`, clear `activeQuestId` if it matched, re-render.
- `selectQuest(id)` → set `activeQuestId`, re-render sidebar (update
  active row) and detail.
- `updateQuestField(id, field, val)` → standard pattern (matches rumors/
  events). For non-display fields, no re-render needed beyond `save()`.
- `setQuestStatus(id, status)` → set status; if the quest's group
  changed (i.e. moved between active and archive groups) AND the quest
  is currently selected, switch sub-tabs to follow it. Save. Re-render
  sidebar + detail.
- `setQuestUrgency(id, urgency)`, `toggleQuestKnown(id)` → standard
  atomic mutators; re-render sidebar (urgency dot may have changed).

**Sidebar rendering.**

- `renderQuestSidebar()`:
  - Filter `state.quests` by sub-tab group: keep quests whose status's
    `group` matches `state.ui.questsSubtab`.
  - If `questStatusFilter` is non-empty, further restrict to that
    status.
  - Apply search filter (case-insensitive substring match against
    title, summary, notes).
  - Apply player-known filter if active.
  - Sort by `URGENCY_ORDER[urgency]` ascending, tiebreak by title
    alphabetic (case-insensitive).
  - Update `q-count` to filtered length.
  - Render rows. Each row shows:
    - Status badge (small colored dot or letter, using
      `Q_STATUSES.color`)
    - Title (or `Untitled`)
    - Urgency dot
    - Location hint (placeholder for now; real value lands in stage 2)
    - Faction color dot if `factionId` set (placeholder until stage 2
      links exist)
    - Lock glyph if `!playerKnown`
- Empty state: `'No quests match these filters.'`

**Status pills bar.** `renderQuestStatusPills()`:
- Pill 0: `All` (active when `questStatusFilter === ''`)
- Pills 1..N: one per status in current sub-tab's group, colored to
  match `Q_STATUSES.color`, active when `questStatusFilter === key`.
- Click toggles: clicking the active pill clears the filter; clicking
  another pill sets it.

**Detail pane (this stage's slice).**

- `renderQuestDetail()` matches the existing event-detail layout shape.
  This stage covers:
  - Header: title input (full width, large), urgency pills, player-known
    toggle, urgency dot, delete button.
  - Body: status pills (5 status options across both groups — the user
    can change between any two), summary textarea.

Stages 2 and 3 fill in link selectors, reward/notes, and event history.

**CSS.** Reuse existing patterns where possible:
- `.quests-pane` mirrors `.factions-detail-pane` (display: flex; flex: 1;
  overflow: hidden).
- `.q-filter-bar`, `.q-search`, `.q-known-btn` — copy from the matching
  `e-` (event) classes.
- `.q-status-pills` and `.q-status-pill` — adapt the
  `.rumor-status-pill` pattern with status-specific colored borders/
  fills.
- `.q-detail-header`, `.q-urgency-pill` — borrow from event-detail; the
  urgency vocabulary is identical.

**Test.**
- Tab opens to empty state on Active sub-tab; `+ New` creates a quest,
  selects it, detail pane renders with empty title, status `available`,
  urgency `medium`. Sidebar shows the new row.
- Switch to Archive sub-tab: empty state (no archived quests yet);
  status pills change to All / Completed / Failed / Abandoned.
- Change selected quest's status to `completed`: sidebar list updates;
  the user is auto-switched to Archive sub-tab; quest stays selected
  and appears in the Archive list.
- Change status back to `active`: switched back to Active sub-tab.
- Status pill filter on Active: clicking `Available` shows only
  available quests. Clicking `Available` again clears the filter.
- Search filter: matches title, summary, notes (case-insensitive). Same
  search field works in both sub-tabs (search results scoped to visible
  sub-tab).
- Player-known toggle: filters list to `playerKnown === true` only,
  scoped to the visible sub-tab.
- Delete quest: confirm dialog; on confirm, removed from list. If it
  had events linked via `questId`, those events still exist with
  `questId === null` — verify by switching to Events tab.
- Reload: active sub-tab restored, status pill filter reset to '' (this
  is intentional — sub-tab persists, sub-filter resets), search and
  known toggle restored, active quest restored.

---

### Stage 2 — Link selectors, reward, notes

End of stage: detail pane is feature-complete except for the event
history readout (stage 3).

**Detail pane additions.** Add below the title/status/urgency/known/
delete header and the summary textarea, above where the event history
will go:

- **Giver NPC selector.** `<select>` of all NPCs. `<option value="">No
  giver</option>` plus all NPCs labeled `name` or `name (faction-name)`
  if applicable, exactly matching the rumor `npcSourceId` selector
  pattern. Sets `giverNpcId`.
- **Sponsoring faction selector.** `<select>` of all factions. Similar
  pattern. Sets `factionId`. When changed, recolor a small accent on
  the detail pane (e.g. left border of the title input) using the
  faction's color, mirroring how rumors colour the card top border.
- **Location row.** Two related controls bound by the pin/hex mutex:
  - Pin selector: `<select>` of all pins (label = pin name + hex
    coords, e.g. `Brackmoor (-3,2)`). `<option value="">No pin</option>`
    at top.
  - Hex selector: `<select>` of all hexes that have a name (label =
    `hex.name (col,row)`) plus a `<option value="">No hex</option>`.
    Hexes without names don't appear in the dropdown — too noisy.
    (Free-typed `col,row` picker is deferred.)
  - **Mutex.** Setting pin clears hex; setting hex clears pin. After
    any change, re-render so the cleared selector reflects "(none)".
    Setting one to its empty option does not clear the other.

- **Reward textarea.** Single-line-ish (small `min-height` ~40px) but
  resizable. Placeholder: `Coin, items, favors, knowledge…`.
- **Notes textarea.** Larger (~120px min-height). Placeholder: `GM
  notes, branches, complications…`.

**Field updates.** Each control calls `updateQuestField` with the right
field name. For status / urgency / playerKnown, keep the dedicated
atomic mutators from stage 1.

**Sidebar location hint.** With link selectors live, the row's "location
hint" is now real:
- If `pinId` set and pin exists → pin name.
- Else if `hexKey` set and that hex has a name → hex name.
- Else if `hexKey` set → coords e.g. `(-3,2)`.
- Else blank.

**Faction color dot.** If `factionId` is set and faction exists, render
the faction's color dot in the sidebar row.

**Test.**
- Set giver to an NPC. Reload. Selector preserves selection.
- Set faction to one with a color. Title accent reflects faction color.
- Pin/hex mutex: set pin → hex selector resets to "(none)". Set hex →
  pin selector resets. Set both to "(none)" → both blank.
- Sidebar row's location hint reflects current pin/hex.
- Faction dot in sidebar row updates when faction changes.
- Edit reward and notes; persist across reload.
- Search filter matches text in `notes` (verified by typing a unique
  string into notes and searching for it).

---

### Stage 3 — Event history, cascade audit, polish

End of stage: slice complete and ready for slice 18 to bolt cross-link
UI on top from the NPC, rumor, and pin sides.

**Event history readout.** Below the notes textarea in the detail pane:

- Section title `Event History` (Cinzel small caps, matching existing
  section-title style).
- Filter `state.events` by `e.questId === q.id`. Sort by event `date`
  string descending — string comparison; freeform dates are accepted by
  design.
- Each row: date (or `(no date)` muted), urgency dot, first 60 chars of
  text, lock glyph if hidden, and a `→` button on the right. Click the
  row OR the arrow → switch to Events tab and select that event:
  `state.ui.activeTab = 'events'; state.ui.activeEventId = e.id; save();
  switchTab('events');` then call the events-tab render functions.
- Empty state: `No events linked to this quest yet.` muted italic.
- Below the list, a small note: `To add an event, open the Events tab
  and link it to this quest from there.` (Bidirectional creation is
  slice 18 territory.)

**Cascade audit.** Verify each of these is implemented; add any missing:

1. **Delete quest** → `state.events.forEach(e => { if (e.questId === id)
   e.questId = null; })`. Implemented in `deleteQuest` (stage 1 already
   specs this).
2. **Delete faction** → set `factionId = null` on all quests. Audit
   `deleteFaction`; add the `quests` line to its cascade if missing.
3. **Delete NPC** → set `giverNpcId = null` on all quests. Audit
   `deleteNpc`; add if missing.
4. **Delete pin** → set `pinId = null` on all quests. Audit pin
   deletion (added in slice 15); add if missing.
5. **Delete rumor** → no quest cascade (rumors don't reference quests
   directly). The TODO comment in `deleteRumor` referencing
   `events.pinIds / events.questId` is misleading — rumors don't touch
   either. Remove the TODO comment.
6. **Delete hex** (resize) → quests with a `hexKey` matching a deleted
   hex should set `hexKey = null`. Audit the hex-deletion path in
   `applyMapSettings` (or wherever hex deletion happens). Add if
   missing.
7. **Delete event** → no cascade (events are leaves; quests don't
   reference event ids).

After auditing, update `data-model.md` only if a documented cascade was
incorrect.

**Polish:**

- Toast on quest creation: `New quest created`.
- Toast on quest deletion: `Quest deleted`.
- Toast on auto-subtab-switch when a quest moves between Active and
  Archive: `Moved to Archive` / `Restored from Archive`.
- Quest count in sidebar header reflects the filtered, current-sub-tab
  count (matches behavior of other sidebars).
- `escHtml` audit: every interpolated user string (title, summary,
  reward, notes, search input value, all selector option labels) is
  escaped. Audit stage 1 and 2 templates.

**Documentation.**

- `data-model.md`:
  - No schema change. Quest shape unchanged.
  - Add a short note on the rendering rule that Active sub-tab shows
    `available + active` and Archive sub-tab shows the rest. (One
    sentence, in the Quest section.)
- `ROADMAP.md`: move slice 16 to Done. Add a short note that quests now
  have full UI with Active / Archive sub-tabs, and that pin / NPC /
  rumor → quest cross-links are still pending in slice 18.

**Test.**
- Quest with 3 events linked via questId: event history shows 3 rows.
  Click a row → Events tab opens, that event is selected.
- Quest with 0 events: empty state placeholder shown.
- Delete a faction sponsoring 2 quests: both quests' `factionId`
  becomes null; faction selectors show `(none)`.
- Delete an NPC giver: `giverNpcId` clears on all quests where it was
  set.
- Delete a pin: `pinId` clears on all quests anchored to it.
- Resize map to delete a hex with a quest's `hexKey`: that quest's
  `hexKey` becomes null.
- Move a quest Active → Archive (mark completed): if it's selected,
  sub-tab auto-switches; toast appears.
- Move a quest Archive → Active (mark active again): sub-tab switches
  back; toast appears.
- Save and reload: sub-tab preserved, all field values preserved,
  active quest restored, search/known toggle restored, status filter
  resets to '' (intentional).

---

## Out of scope

- Cross-link UI from the NPC / rumor / pin side (slice 18).
- Bidirectional event creation from quest detail.
- Reward parsing into structured fields (it stays freeform).
- Quest dependencies (quest A unlocks quest B).
- Quest tags / categories.
- Player-facing quest log / handouts.
- Time tracking on quests (no clocks).
- Free-typed `col,row` picker for unnamed-hex anchors.
- Bulk operations (mark all completed, etc.).
- Separate `archived` boolean orthogonal to status.

## Files

- `index.html` — full Quests tab, all behavior + CSS.
- `data-model.md` — one-sentence note on Active / Archive rendering
  rule.
- `ROADMAP.md` — move slice 16 to Done.

## Definition of done

- All three stages land cleanly.
- Tab is feature-complete: sub-tabs, CRUD, status filter pills, search,
  known toggle, all field editors, link selectors with pin/hex mutex,
  reward, notes, event history readout.
- Status changes correctly move quests between Active and Archive sub-
  tabs, with auto-switch when the moved quest is selected.
- Every documented cascade involving quests is implemented and tested.
- No schema change; existing slice-15 saves continue to load.
- The Events tab's quest link and the quest detail's event history are
  consistent.
