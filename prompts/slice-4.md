Slice 4: port the Relations matrix into the Relations tab of index.html.
Reference: reference/faction-tracker.html has the existing matrix implementation. Port it into our merged app, adapted to the new data model in docs/data-model.md.
Scope:

The Relations tab shows a square grid of all factions × all factions. Row labels and column headers show faction names, each with the faction's color dot.
Self-cells (diagonal) are visually inert — striped or dimmed, not clickable.
Each off-diagonal cell shows the current relation type using the vocabulary from the data model (Mortal Enemies, Sworn Enemies, Hostile, Tense, Neutral, Friendly, Allied, Sworn Allies, plus None and Unknown).
Single-click a cell to cycle through relation types in the order listed in the data model doc.
Double-click a cell (or a hover-revealed ✎ button) opens a modal where the user can edit the note for that relation.
A playersAware toggle lives on each cell — small pill or icon indicating whether the players know this relationship. Distinct visual from the type cell.
A legend above the grid showing all relation types with their colors.
Empty state: if fewer than 2 factions exist, show a message prompting the user to add more.

The narrative-event rule (important):
When the user changes the type on a relation where playersAware === true, show a confirmation modal with the text specified in docs/data-model.md ("The players know this relationship as <current type>. Changing it to <new type> represents a narrative event..."). The modal has Confirm and Cancel buttons. Confirming applies the change; cancelling leaves the relation unchanged. Toggling playersAware itself never prompts. Editing the note never prompts.
Visual design:
Match the aesthetic of the reference file's matrix — dark cells, colored text/backgrounds per relation type. You'll need to invent color mappings for the new types (Mortal Enemies down through Sworn Allies, plus None and Unknown), since the vocabulary changed. Keep the palette in the same family as the rest of the app.
Data integration:

Relations live in state.relations, keyed by sorted([factionIdA, factionIdB]).join("|").
Default for any unrecorded pair is { type: "unknown", playersAware: false, note: "" }. Don't eagerly populate state.relations on faction create — let entries materialize only when the user first interacts with that cell.
Cascade: when a faction is deleted (in slice 3's code), relations involving it are already cleaned up. Don't duplicate that logic here; just make sure the matrix re-renders correctly after a faction is deleted.

Out of scope:

No map, rumors, quests, events changes.
No cross-linking to other panels.

When done, summarize: the relation type → color mapping you chose, how the narrative-event confirmation is implemented, and what I should test. Keep total added code reasonable (~400-600 lines).