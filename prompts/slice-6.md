Slice 6: add a Players tab with roster management, bump schema to v3.
Reference: reference/session-planner.html has the Player Roster implementation in its "Roster" tab. Port that roster UI into a new Players tab in the merged app, adapted to the data model at docs/data-model.md.

Part 1 — Schema migration (v2 → v3):
In the load() function's migration chain, add a v2 → v3 step that:

Ensures state.players exists (default []).
Ensures state.sessions exists (default []) — leave empty for now, Sessions tab is a future slice.
Adds sessionId: null and playerKnown: false to every existing event (we don't have events yet, but the migration should handle them if they exist).
Sets state.schemaVersion = 3.

Part 2 — Add Players tab:
Add a new top-level tab "Players" between "Relations" and whatever comes last. Update the tab-button row in the header and the content area to include it.
Part 3 — Roster UI:
The Players tab shows a grid of player cards (match the reference/session-planner.html layout — roster-grid with auto-fill, minmax(300px, 1fr) columns).
Each card has:

Color picker row at the top (8 colors from the reference file's PLAYER_COLORS).
Player name input (the human's name).
Four grid fields: Character (in-game name), Class, Level / XP, Current Hex.
Playstyle & Notable Skills textarea.
Notable Skills textarea (skills field from the schema — the reference file has prefs and skills as separate fields; make sure both are present).
Loot & Notable Items list (add/remove rows, single freetext input per row).
Delete button (top-right corner, confirm before deleting).

An "Add Player" card at the end of the grid opens a new empty card.
Part 4 — Current Hex validation:
The "Current Hex" field stores currentHexKey as "col,row" OR null. The input shows the key value. If the user types a key that doesn't exist in state.hexes, show a red-border warning style on the input and a small tooltip or inline note saying "Hex not found on map." Do NOT prevent saving — just flag it visually. If they clear the field, store null.
(Note: slice 7+ will add the map, so right now state.hexes is empty and every non-null value will flag as invalid. That's fine — the validation is structural and will start working correctly once the map exists.)
Part 5 — Styling:
Match the reference file's player-card, player-name-input, field-input, pref-textarea, color-dot-row, etc. Reuse the existing CSS variables from the merged app — don't re-define colors.
Out of scope:

No sessions, no attendance UI (that's slice 7).
No Faction/NPC/Map/Rumors/Quests/Relations changes.
No import from wm_sessions_v1 (deferred).

When done, summarize: (a) what the migration does, (b) a note on the currentHexKey validation behaviour with an empty map, (c) what I should test. Keep added code reasonable (~400-600 lines).