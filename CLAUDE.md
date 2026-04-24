# Western Marches Tracker

A single-page web app for running a Western Marches style tabletop RPG campaign.
Combines a hex-based world map with a faction/NPC/rumor tracker, with data linked across both.

## Current state
This project is merging two existing standalone HTML tools:
- `reference/faction-tracker.html` — factions, NPCs, rumors, faction-relations matrix
- `reference/hexmap.html` — hex world map, per-hex local maps with pins

These reference files are READ-ONLY. Do not modify them. The merged app lives in
`index.html` at the project root (create it when needed).

## Entities (see docs/data-model.md for full schema)
Factions, NPCs, Rumors, Quests, Events, Players, Sessions, Hexes, Pins, Relations.
All top-level collections keyed by opaque IDs. Cross-references use IDs only.

## Architecture constraints
- Single HTML file. All CSS inline in `<style>`, all JS inline in `<script>`.
- Vanilla JS only. No frameworks, no build step, no npm, no bundler.
- No external runtime dependencies at all. Fonts are bundled locally in `fonts/`. The app must work fully offline from `file://`.
- Persistence is `localStorage` under a single key: `wm_unified_v1`.

## Data model
The merged data model lives in `docs/data-model.md`. Read it before making changes
that touch state shape. If a change requires altering the schema, update that doc
in the same commit.

## Style
- Fonts: Cinzel (headings) and Crimson Pro (body) — bundled locally in `fonts/`.
- Color palette matches the reference files: dark parchment, gold accents, muted earth tones.
- Reuse the CSS variables from the reference files where possible.

## Workflow
- Work in small slices. One feature, one commit.
- Before large changes, propose a plan in chat and wait for confirmation.
- After edits, summarize what changed and what to test in the browser.
- Never delete or rewrite large sections without explaining why first.

## Out of scope (for now)
- Multi-user sync or any backend
- Export/import beyond plain JSON
- Mobile-responsive layout (desktop only)
- Undo/redo beyond what git provides

## Planned future features (don't build yet, but don't preclude)
- NPC detail page with GURPS stats, traits, profession, appearance
- NPC-to-NPC relationships (separate from faction relations)
- PC-to-NPC relationships (PCs may become a separate top-level collection)
- Designs that hardcode NPCs as "just a name + role" will need to be revisited