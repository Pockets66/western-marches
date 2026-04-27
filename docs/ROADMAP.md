# Roadmap (as of slice 14 complete)

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

## In progress


## Planned
- Slice 14.5: Travel time calculator helper
- Slice 15: Local maps + pins
- Slice 16: Quests tab
- Slice 17: Cross-link pass 1 (pin ↔ faction, pin ↔ NPCs)
- Slice 18: Cross-link pass 2 (rumor ↔ pin/hex, NPC ↔ pin, session autocomplete)
- Slice 19: Narrative-event recorder + map tweaks
- Slice 20: Character sheet Tier 2 (structured advantages/disadvantages/skills)
- Slice 21: Character sheet Tier 3 (combat, equipment, defenses)

## Deferred / may skip
- Spells sub-tab on character sheets (Tier 4, unmentioned so far)

## Working agreements
- Linear commits on main, no branching (unless a slice gets experimental)
- Stage-based slices with explicit "go" confirmation between stages
- You commit, not Claude Code (Workflow A) — though slice 9-10 drifted to Workflow B
- prompts/slice-N.md file per slice, committed before the slice starts
- localStorage backup before any schema-changing slice
- Two-terminal setup: Tab 1 for Claude Code, Tab 2 for git
