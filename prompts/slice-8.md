First, a one-shot "here's the context" message that sets up all four stages at once:

We're doing Slice 8: add a Sessions tab to the merged app, using the Session entity from docs/data-model.md.
Method: surgical additions to index.html, not a rewrite. String-replace style edits only. If you modify existing code, call it out.
We'll work in four stages. I'll send them one at a time. For each stage: implement, summarize what you did, then wait for me to test and give you the next stage. Do not proceed to later stages on your own.
Styling throughout: match the existing app — section-box pattern, Cinzel labels, Crimson Pro body, gold accents.
Out of scope for the whole slice: no cross-linking to factions/NPCs/hexes/pins/rumors, no events system, no import from old localStorage keys, no export.
Ready for Stage 1?

Wait for it to acknowledge, then feed each stage individually. Keep each stage prompt short and focused:
Stage 1

Stage 1 — Scaffolding.

Add state.ui.activeSessionId to the state shape (default null).
Verify state.sessions exists on load; if records are missing expected fields, default them to "" or [] as hygiene.
Add a "Sessions" tab button to the header tab bar.
Add the content panel with sidebar + detail layout.
Placeholder message in the detail area: "No session selected."

No CRUD yet, no forms. Just the shell.
Summarize what you changed.

Stage 2

Stage 2 — Session CRUD.

"+ New Session" button. Creates a session with a fresh id and all fields defaulted empty.
Sidebar list: title + date below it + attendance count chip, newest first, active highlighted, click to select, persist activeSessionId.
Detail view header strip: title input, date input (freeform text, not a date picker), playerVisible toggle, delete button with confirm.

No other sections yet.
Summarize.

Stage 3

Stage 3 — Attendance + three content layers.

Attendance chip grid: one chip per player in state.players, each showing color dot + character name (fallback to player name, then "Unnamed"). Click to toggle presence. Empty-state message if zero players.
Planning Layer (collapsible, expanded by default): Active Hooks list of {text, source} rows with add/remove; Planned textarea.
Play Layer (collapsible, expanded by default): Actual Play Notes textarea.
Narrative Layer (collapsible, collapsed by default): Location input, Report textarea, GM Notes textarea. Style Report and GM Notes distinctly from the planning/play textareas.

Collapse state in memory per view, not persisted.
Summarize.

Stage 4

Stage 4 — Tagged lists + player-delete cascade.

Five collapsible tagged list sections, all in a final "Tagged Lists" block (collapsed by default):

Hexes Visited (single-input rows)
Factions/NPCs Encountered ({text, sub} rows)
Loot ({text, sub} rows)
Casualties ({text, sub} rows)
Misc ({text, sub} rows)


Generic addListItem, updateListItem, removeListItem helpers parameterized by list name.
Cascade: when a player is deleted from the Players tab, remove their entry from every session's attendance object. This modifies the existing deletePlayer logic — flag it explicitly in your summary.

Summarize everything I should test across all four stages.