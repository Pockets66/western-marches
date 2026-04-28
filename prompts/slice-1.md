Read CLAUDE.md and docs/data-model.md. Then create index.html at the project root containing:

The tab shell: header with five tab buttons — Map, Factions, Rumors, Quests, Relations. Clicking a tab switches which panel is visible. Each panel is a placeholder div containing just the tab's name in brackets.
The state layer: a state object matching the top-level shape from the data model doc, initialized empty. save() writes to localStorage key wm_unified_v1. load() reads and hydrates. uid() generates IDs. Include schemaVersion: 1.
CSS foundation: pull the color variables, Google Fonts imports (Cinzel + Crimson Pro), base typography, and tab styling from reference/faction-tracker.html. Match that file's look.

Do NOT port any panel content, map rendering, faction CRUD, or anything beyond the shell. This is slice 1 of a multi-slice merge. Keep it under 400 lines.
When done, summarize what you built and what I should test in the browser.