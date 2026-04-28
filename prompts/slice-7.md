Slice 7: Factions reorganization + Player character sheet Tier 1.
This slice has two parts. Do them in order and commit them together only once both pass testing.

PART A — Factions reorganization

Remove the "Relations" top-level tab from the header tab bar. Its code (the matrix, the cycle/click behaviour, the narrative-event confirmation modal, the legend) stays — we're moving it, not deleting it.
Inside the Factions tab content area, add a sub-tab row at the top with two buttons: "Detail" and "Relations." Style these as smaller/secondary compared to the top-level tabs (a visually distinct style so the user can tell main-tab vs. sub-tab at a glance).
"Detail" sub-tab renders the existing faction sidebar + detail view (the entire current Factions tab content).
"Relations" sub-tab renders the relations matrix (legend + grid + modal). The empty-state message ("Add at least two factions...") still applies.
The sub-tab state is session-only — no need to persist ui.factionsSubtab. Default to "Detail" on page load.
Update state.ui.activeTab type to no longer include "relations" as an option. Any persisted value of "relations" on load should be migrated to "factions".


PART B — Player schema migration and Tier 1 sheet
Schema migration (v3 → v4):
For each player in state.players:

Rename the existing skills field to notes (freeform GM notes).
Delete the level field.
Keep cls (will be labeled "Profession / Template" in the UI).
Add these new fields, all defaulting to empty string "":
pointTotal, unspentPoints, st, dx, iq, ht, hp, will, per, fp, basicSpeed, basicMove, dodge, parry, block, dr, thrust, swing, advantages, disadvantages, skills.

Set schemaVersion = 4. Update docs/data-model.md Player entity block and migration notes to match.
Players tab layout change:
The current Players tab is a grid of cards. Change it to the same pattern the Factions tab uses: sidebar list on the left, detail view on the right. The sidebar shows player name (or "Unnamed") with a small color dot; the active player is highlighted. A "+ Add Player" button lives at the top of the sidebar. The empty state (no player selected) shows in the detail area.
Use state.ui.activePlayerId to track selection. Persist it.
Tier 1 character sheet in the detail view:
The detail view is a structured form, styled to match the existing app language (section-box patterns, Cinzel labels, Crimson Pro body, gold accents). Don't try to mimic the visual layout of the GURPS PDF — build it as tasteful dark-panel sections.
Sections, in this order:

Identity — Player name, Character name, Profession / Template (from cls), Point Total, Unspent Points, Color picker row, Current Hex (with the validation from slice 6 — red border if not a valid hexKey, but doesn't block saving).
Attributes — Two columns. Left column: ST, DX, IQ, HT as labelled inputs. Right column: HP, Will, Per, FP, Basic Speed, Basic Move. Show small helper text under each secondary stat showing the default formula in grey (e.g. "= ST" under HP, "= (HT+DX)/4" under Basic Speed). All fields are string inputs; no computation.
Defenses & Damage — One row: Dodge, Parry, Block, DR. Below it: Thrust, Swing. All string inputs.
Advantages / Perks — Multi-line textarea. Placeholder: "One per line. E.g. Combat Reflexes [15]".
Disadvantages / Quirks — Multi-line textarea. Placeholder: "One per line. E.g. Bad Temper [-10]".
Skills — Multi-line textarea. Placeholder: "One per line. E.g. Brawling-14 [4]".
Playstyle & Notes — Two textareas side by side: "Playstyle" (the prefs field) and "GM Notes" (the notes field, which was previously skills).
Loot & Notable Items — the existing list UI from the roster, preserved.
Delete Player button at the bottom, confirm before acting.

Data binding:
Every input calls a single generic updatePlayerField(id, field, value) that writes to the player record and calls save(). Don't write 20 one-off handlers.

Out of scope:

No languages, senses, reaction modifiers, wealth/status, melee/ranged attack tables, equipment list, spells, current-HP tracker, point breakdown, encumbrance table, or auto-computation. All Tier 2+.
No map, no sessions, no events.
No changes to Factions internals beyond the sub-tab wrapper.


When done, summarize:
(a) the migration steps,
(b) how the sub-tab pattern is styled vs. the main tab bar,
(c) any place where Relations-related code needed adjustment when moved,
(d) screenshot-worthy sections of the character sheet layout in description,
(e) what I should test.
Keep added code reasonable. This is a larger slice than prior ones, but it shouldn't exceed ~1500-2000 lines of additions. If it looks like it'll blow past that, pause and check in.