Slice 3: port the Factions panel into index.html.
Reference: reference/faction-tracker.html has the existing standalone implementation. Port its Factions functionality into our merged app, adapted to the new data model in docs/data-model.md.
Scope:

Sidebar with faction list on the left of the Factions tab (matching the reference's layout).
Faction detail view on the right: name, color picker, threat pills, playerKnown toggle, goals, notes, agenda clock (all 5 sizes, click-to-advance), and a Key NPCs section.
Create / select / delete factions. Create / edit / delete NPCs within the faction view.
All mutations go through save() to persist in wm_unified_v1.

Key differences from the reference implementation — these matter:

NPCs are stored in state.npcs as a flat array with their own ids, NOT nested inside the faction. The Faction detail view shows NPCs where npc.factionId === activeFaction.id. Creating an NPC from the faction view sets its factionId to the active faction.
NPCs have pinId: null for now (we'll wire up pin linking in a later slice).
Deleting a faction follows the cascade rule in docs/data-model.md: set factionId = null on the faction's NPCs (do NOT delete them), and delete all state.relations entries involving this faction.
Deleting an NPC follows its cascade rule (we don't have rumors/quests/events yet, so it's a no-op for now — but structure the delete handler so cascade logic is easy to add later).
Do NOT port the interactions array (faction event log) or the per-NPC log array. Those are superseded by state.events in a future slice. Just leave the Event Log section out of the faction detail for now.

Out of scope:

Rumor Board, Relations matrix, Quests tab, Map tab — untouched.
No cross-linking to pins, quests, rumors, or events.
No migration from old localStorage keys.

When done, summarize: what's in the Factions panel, what cascade rules you implemented, and what to test. Keep total file size under ~1500 lines.