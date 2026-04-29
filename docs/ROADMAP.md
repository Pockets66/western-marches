# Roadmap (as of slice 16 complete)

## Done
- Slice 1: Shell + state layer + CSS foundation
- Slice 2: Local fonts, offline-capable
- Slice 3: Factions panel (flat NPCs in data model)
- Slice 4: Relations matrix (GURPS long-term posture vocabulary)
- Slice 5: Faction schema refactor (alignment, activity, size, multiple clocks)
- Slice 6: Players roster (schema v3)
- Slice 7: Factions sub-tabs + Tier 1 character sheet (schema v4)
- Slice 8: Sessions tab (consolidated session+expedition, 4 stages)
- Slice 9: World hex map (grid, terrain, hex metadata, schema v5)
- Slice 10: Signed coord system + directional expansion (schema v6)
- Slice 11: Rumors board with archive (schema v7)
- Slice 12: NPCs tab (schema v8)
- Slice 13: Events system (schema v9)
- Slice 14: Map overlays — rivers and roads (schema v10)
- Slice 14.5: Travel time calculator + route overlays (schema v11)
- Slice 14.7: Smooth overlay rendering + Bezier curves + bridge semantics
- Slice 14.8: Network-smoothed Catmull-Rom rendering + water overlays + multi-source rivers + trunk overrides (schema v12)
- Slice 14.9: Dropped — center-origin support subsumed by slice 14.10 segment model
- Slice 14.10: Segment-based overlay model consolidation (schema v13) — smooth turns, parallel roads, segment editor, sync flow, lake-blocks-roads
- Slice 14.10.1: Overlay rendering fixes — ponds no longer erase rivers, centripetal Catmull-Rom, no center-snapping on drag, per-segment flow arrows
- Slice 14.11: Map undo system — snapshot-based, in-memory, 50-entry cap, drag = one unit
- Slice 15: Local maps + pins (schema v14) — pin CRUD, world-map dots, local map modal, image upload, drag/edit, cascades
- Slice 16: Quests tab — Active/Archive sub-tabs, status/urgency, all field editors, event history readout; cross-link UI pending slice 18

## Deferred

## In progress


## Planned
- Slice 17: Cross-link pass 1 (pin ↔ faction, pin ↔ NPCs)
- Slice 18: Cross-link pass 2 (rumor ↔ pin/hex, NPC ↔ pin, session autocomplete)
- Slice 19: Narrative-event recorder + map tweaks
- Slice 20: Character sheet Tier 2 (structured advantages/disadvantages/skills)
- Slice 21: Character sheet Tier 3 (combat, equipment, defenses)

## Deferred / may skip
- Spells sub-tab on character sheets (Tier 4, unmentioned so far)

## Planned Patches / Features
- Recolour Hex Title fonts for better visibility
- Recolour roads for better visibility (add yellow border)
- Double-click hex opens local map
- Draw/Edit travel route
- Routes crossing a river add appropriate travel time (each crossing = +0.25 days of travel IF no bridge)
- Tiles with a "lake" feature add 0.25 days of travel to routes passing through opposite edges.
- Route calculator finds shortest possible travel route WITHOUT going through impassable hexes
- Route shown on map with dashed line by default -> "Mark Route on Map" button changes to "Save Route on Map"
- Toggle explored filter in map mode (GM Only)
- Player-facing version of map
- Player-facing version of each tab
- Toggle local map pin visibility on world-map
- Map Settings menu where terrain travel modifiers and other defaults can be edited
- Port everything to web with live updating player-facing pages
- Player forum

## Working agreements
- Linear commits on main, no branching (unless a slice gets experimental)
- Stage-based slices with explicit "go" confirmation between stages
- You commit, not Claude Code (Workflow A) — though slice 9-10 drifted to Workflow B
- prompts/slice-N.md file per slice, committed before the slice starts
- localStorage backup before any schema-changing slice
- Two-terminal setup: Tab 1 for Claude Code, Tab 2 for git
