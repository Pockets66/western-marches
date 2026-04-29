# Slice 18 — Cross-link Pass 2 (rumor ↔ pin/hex, NPC ↔ pin, quest polish, session autocomplete)

## Goal
Continue the cross-link surfacing started in slice 17. Wire up rumor-to-place
links, give NPCs a real pin picker, fulfill the "cross-link UI pending slice
18" promise on slice 16's Quests tab (jump-buttons + reverse views), and add
autocomplete suggestions to session tagged lists. All UI; no schema change.

## Non-goals
- Session tagged lists remain **free-text storage**. Autocomplete suggests
  against real records but the stored value stays a string. (Per
  `data-model.md`: "Automatic link resolution on session tagged lists (stays
  free text)" is out of scope for v1.)
- No new pin↔* links beyond NPC. Pin↔faction and pin↔NPCs were covered in
  slice 17.
- No schema change. No localStorage backup needed.

## Schema
No changes. Schema stays at v14.

Field reminders:
- `Rumor.pinId: string | null` and `Rumor.hexKey: string | null` already
  exist with the validation rule "exactly one set, OR neither" (UI-enforced).
- `NPC.pinId: string | null` already exists.
- `Quest.pinId / hexKey / factionId / giverNpcId` all exist; pickers shipped
  in slice 16. What's missing is jump-out navigation and reverse views.
- Session tagged lists: `hexesVisited: [string]` and
  `factionsEncountered/loot/casualties/misc: [{text, sub}]`. Stays as-is.

## Stages

### Stage 1 — Rumor card pin/hex picker
The rumor card already shows urgency, status, faction, and NPC source. Add a
location row that mirrors **the slice-16 quest pattern** for consistency: two
side-by-side dropdowns (one for pin, one for hex) with mutual-exclusion logic
and a hint line.

- Two `<select>` controls in a single row labelled "Location":
  - Pin select: `<option value="">No pin</option>` then every pin labelled
    `pin.name (hexKey)` (mirrors `pinOpts` in `renderQuestDetail`).
  - Hex select: `<option value="">No hex</option>` then every **named** hex,
    sorted by name (mirrors `hexOpts` in `renderQuestDetail`).
- Hint line below: "Setting one location clears the other." (Same wording as
  the quest detail.)
- New writers `setRumorPin(id, val)` and `setRumorHex(id, val)` parallel to
  `setQuestPin` / `setQuestHex` — set one, clear the other, save, re-render
  the rumor card.
- Compact location tag on the rumor card (small enough to fit alongside the
  faction stripe): 📍 Pin name, or ⬡ hexKey (or hex.name if set). Click the
  tag to jump — pin-tag opens the local map with the pin selected; hex-tag
  switches to the Map tab and selects that hex.

Acceptance:
- Selecting a pin clears any previously set hex, and vice versa.
- Selecting "No pin" / "No hex" works as expected; both empty leaves the
  rumor unanchored.
- The "exactly one OR neither" rule is never violated.
- Tag-click navigates correctly.
- Existing pin-delete cascade nulls `r.pinId`; verify the card re-renders
  cleanly.

### Stage 2 — NPC detail pin picker
Replace the read-only pin display introduced in slice 17 stage 3 with a real
picker.

- Single-select dropdown directly below the Faction picker; same `iSel`
  styling as slice 17 stage 1.
- Options: `<option value="">— mobile / unknown —</option>` then every pin
  labelled `pin.name (hexKey)`.
- Writes `n.pinId` on change. Re-render NPC sidebar so the location tag
  updates.
- Keep the "Open" button from slice 17 stage 3.
- In the NPC sidebar row, show a small location tag with the pin name
  (truncate if needed) so the NPC list is browsable by location.

Acceptance:
- Picker reflects the current `pinId`.
- Setting and clearing both persist and update the sidebar.
- Pin deletion correctly nulls `pinId` (existing cascade).

### Stage 3 — Quest cross-link polish (the "pending" half from slice 16)
Slice 16 shipped quest pickers but no jump-out navigation and no reverse
views. Close that gap.

**Jump-buttons on quest detail.** Next to each picker in `renderQuestDetail`,
add a small "→" button (or make the selected label itself clickable) that
navigates to the chosen entity:

- Giver NPC → `goToTab('npcs', { activeNpcId })`.
- Faction → `goToTab('factions', { activeFactionId })`.
- Pin → opens the local map for the pin's hex with the pin selected.
- Hex → switches to the Map tab and selects that hex.
- Disable / hide the jump button when the corresponding field is null.

**Quest reverse views.** Add quest rollups to the entities a quest references:

- **NPC detail**: "Quests offered" section listing every quest with
  `q.giverNpcId === n.id`. Each row: status pill · title · urgency dot.
  Click → goToTab quests with `activeQuestId` set.
- **Faction detail**: "Quests" section parallel to "Key NPCs" and (from
  slice 17) "Locations". Lists every quest with `q.factionId === f.id`.
  Same row format and click behaviour.
- **Pin editor (in local map modal)**: "Quests at this location" section
  listing every quest with `q.pinId === pin.id`. Same row format and
  click behaviour.
- **World-map hex info panel**: small one-liner "N quests anchored here"
  with click-through (only shown when count > 0). Anchored = `q.hexKey ===
  hexKey` OR a pin in this hex has `q.pinId`.

For all reverse views: skip archived quests by default (status in
`completed/failed/abandoned`), match what the Active sub-tab shows. Optional
toggle to include archived if a stretch goal — defer if it bloats the stage.

Acceptance:
- Every jump-button on quest detail navigates correctly when the field is
  set; renders disabled / absent when null.
- Each reverse view lists the right quests in real time.
- Deleting a quest removes it from all reverse views without breaking
  anything (existing delete handler nulls `event.questId`; reverse views are
  derived, no extra cascade needed).
- Archived quests are excluded by default.

### Stage 4 — Session tagged-list autocomplete
Add suggestions to the existing session tagged-list inputs without changing
how data is stored.

- Use a `<datalist>` per input — cheap, native, no popup engine to write.
- One shared datalist per list type, populated from real records:
  - `hexesVisited` main field → datalist of every hex key, label
    `key · name` if named.
  - `factionsEncountered` main field → datalist of faction names.
  - `factionsEncountered` sub field → datalist of all NPC names.
  - `loot` main field → no autocomplete (free text wins).
  - `casualties` main field → datalist of NPC names + every player's
    `character`.
  - `misc` → no autocomplete.
- Storage stays free text. If the user types something that matches a record,
  that's a coincidence — we don't store IDs.
- Optional polish: when the typed text exactly matches a known record name,
  show a subtle indicator (a faint dot or border colour) hinting "this
  matches a real entity". Pure cosmetic, no behaviour change.

Acceptance:
- Typing in the relevant fields shows native browser suggestions.
- Selecting a suggestion writes the text to the field; the stored object is
  still `{text, sub}` strings.
- Existing session data with arbitrary free-text values continues to work —
  no migration.
- Datalists rebuild fresh whenever a session is opened (don't cache stale
  lists across faction/NPC edits in the same browser session).

### Stage 5 — Polish & link badges
Small UX touches now that all the cross-links exist.

- On the rumor card, if a pin is set, show the faction colour-stripe of the
  pin's faction (if any) on the location tag.
- On a pin in the local map, render a small badge with the count of NPCs,
  rumors, and quests at that pin (`n.pinId === pin.id`, `r.pinId === pin.id`,
  `q.pinId === pin.id`). Helps the GM spot "busy" pins at a glance.
- On the world map info panel for a hex, list a one-liner of pins in that
  hex with NPC/rumor/quest counts each.

Acceptance:
- Badges update live as cross-links are added/removed.
- Empty cases render as nothing (no "0 NPCs" clutter).

## Test plan (final manual smoke)
1. Create a rumor. Pick a pin → confirm `pinId` set, `hexKey` null. Switch to
   a hex → confirm `pinId` null, `hexKey` set. Pick none → both null.
2. Click the rumor's location tag → navigates to the right map.
3. From NPC detail, set a pin via the picker → confirm sidebar shows the pin
   name. Open button works.
4. Open a quest with a faction, giver NPC, and pin set. Click each jump
   button → lands on the right tab with the right entity selected. Click the
   pin jump → opens the local map with the pin highlighted.
5. From an NPC who gives a quest, confirm "Quests offered" lists it. From
   that quest's faction, confirm "Quests" section lists it. From the pin in
   the local map, confirm "Quests at this location" lists it.
6. Mark the quest "completed" → confirm it disappears from the reverse views
   (default-active filter).
7. Delete the pin → rumor's location clears, NPC's pin clears, quest's pin
   clears, all reverse views update, no broken UI.
8. In a session, type in `factionsEncountered` main → see real faction names
   suggested. Type in sub → see NPC names. Type in `hexesVisited` → see hex
   keys.
9. Type a totally unknown faction name in the session field → confirm it
   saves as free text.
10. Visual: pin in local map shows correct NPC/rumor/quest count badges.

## Out of scope (deferred)
- Promoting session tagged lists to typed references — explicitly deferred
  per `data-model.md`.
- Hex-only "events at this hex" rollup — the Events tab already filters by
  hex.
- Inline editing of cross-links from the world-map info panel — possibly
  slice 19.
- Including archived quests in reverse views by default — TBD; current plan
  is to hide them.

## Workflow
- Commit `prompts/slice-18.md` first.
- Stage-based: explicit "go" confirmation between stages.
- One commit per stage. (Stage 3 may want a sub-commit per reverse view if
  the diff gets large.)
- Update `ROADMAP.md` (slice 18 → Done) at the end. Also remove the "cross-link
  UI pending slice 18" qualifier from the slice 16 line.
- Update `data-model.md` only if anything in the schema actually changes (it
  shouldn't).
