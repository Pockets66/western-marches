# Data Model

Single source of truth for the unified app. Any schema change must update this
doc in the same commit as the code change.

## Storage

All state lives in `localStorage` under one key: `wm_unified_v1`.
On load, the app reads that key and hydrates a single in-memory `state` object.
On every mutation, the app serializes `state` back to the same key.

The `v1` suffix is deliberate. If we ever need a breaking schema change, we write
a migration from `wm_unified_v1` to `wm_unified_v2` rather than silently
corrupting existing data.

## Top-level shape

```js
state = {
  schemaVersion: 1,
  factions:  [Faction],
  npcs:      [NPC],
  rumors:    [Rumor],
  quests:    [Quest],
  events:    [Event],
  hexes:     { "col,row": Hex },   // keyed by coord string
  pins:      [Pin],
  relations: { "factionIdA|factionIdB": Relation },  // sorted key
  ui: {
    activeTab: "map" | "factions" | "rumors" | "quests" | "relations",
    activeFactionId: string | null,
    activeQuestId:   string | null,
    activeHexKey:    string | null,   // "col,row"
    rumorFilter: string,
    questFilter: string,
  }
}
```

## IDs

- Every entity (faction, npc, rumor, quest, event, pin) has a string `id`
  generated on creation.
- IDs are opaque — never parse them, never display them to the user, never rely
  on ordering.
- Cross-references use IDs only. Never embed a copy of one entity inside another.
- Generation: `Date.now().toString(36) + Math.random().toString(36).slice(2,6)`.

## Shared enums

```js
// Used on rumors, quests, events.
Urgency = "low" | "medium" | "high" | "critical"
```

## Entities

### Faction

```js
{
  id: string,
  name: string,
  color: string,               // hex, e.g. "#c8972a"
  threat: "dormant" | "rising" | "active" | "defeated",
  goals: string,
  notes: string,
  playerKnown: boolean,
  clock: {
    size: 4 | 6 | 8 | 10 | 12,
    filled: number,            // 0..size
    label: string,
    consequence: string,
  },
}
```

Faction-level session history is no longer stored here. To get a faction's
event history, filter `state.events` by `factionIds`.

### NPC

```js
{
  id: string,
  name: string,
  role: string,
  notes: string,
  factionId: string | null,    // null = independent / freelance
  pinId:     string | null,    // where they're typically found; null = mobile/unknown
}
```

Per-NPC interaction history is stored in `state.events`, filtered by `npcIds`.

### Rumor

```js
{
  id: string,
  text: string,
  status: "unverified" | "confirmed" | "false",
  urgency: Urgency,
  playerKnown: boolean,
  factionId:   string | null,   // which faction this rumor concerns
  npcSourceId: string | null,   // which NPC told the players
  // Exactly one of these is set, or both are null:
  pinId:  string | null,        // tied to a specific location
  hexKey: string | null,        // tied to a hex but no specific pin
}
```

Validation rule: a rumor has `pinId` set, OR `hexKey` set, OR neither.
Setting one clears the other. Enforced in the UI, not the schema.

### Quest

```js
{
  id: string,
  title: string,
  summary: string,
  status: "available" | "active" | "completed" | "failed" | "abandoned",
  urgency: Urgency,
  playerKnown: boolean,
  giverNpcId: string | null,   // who offered it
  factionId:  string | null,   // sponsoring faction, if any
  pinId:      string | null,   // anchor location
  hexKey:     string | null,   // anchor hex (if no specific pin)
  reward: string,
  notes:  string,
}
```

A quest's history is captured by events linked via `questId`.

### Event

A thing that happened at a session, linkable to any combination of entities.

```js
{
  id: string,
  date: string,                // "Session 12", "1/15", whatever convention you use
  text: string,
  urgency: Urgency,
  factionIds: [string],        // zero or more
  npcIds:     [string],
  pinIds:     [string],
  hexKeys:    [string],
  questId:    string | null,
}
```

Events replace the `interactions` array previously on factions and the `log`
array previously on NPCs. "Faction X's history" is now a derived filter, not
stored data.

### Hex

Keyed by `"col,row"` in `state.hexes`.

```js
{
  terrain: "unknown" | "plains" | "forest" | "dark-forest" | "hills"
         | "mountain" | "water" | "swamp" | "desert" | "ruins" | "settled",
  name: string,
  note: string,
  explored: boolean,
  overlays: [Overlay],
}
```

Pins are NOT nested in hexes. Filter `state.pins` by `hexKey`.

### Overlay

```js
{
  type: "river" | "road",
  edges: [number],             // 1 or 2 edge indices, 0..5
  flowFrom: 0 | 1,             // which edge in `edges` is the source (rivers only)
}
```

### Pin

```js
{
  id: string,
  hexKey: string,              // "col,row", which hex it lives in
  x: number,                   // local map coords, pixels
  y: number,
  type: "settlement" | "dungeon" | "ruin" | "landmark" | "threat" | "camp" | "other",
  name: string,
  notes: string,
  discovered: boolean,         // true = players know about it
  factionId: string | null,    // who controls this location
}
```

A pin's NPCs:    `state.npcs.filter(n => n.pinId === pin.id)`
A pin's rumors:  `state.rumors.filter(r => r.pinId === pin.id)`
A pin's quests:  `state.quests.filter(q => q.pinId === pin.id)`
A pin's events:  `state.events.filter(e => e.pinIds.includes(pin.id))`

### Relation

Keyed by `sorted([factionIdA, factionIdB]).join("|")` in `state.relations`.

Relation types follow the GURPS reaction table:

```js
{
  type: "disastrous"    // 0 or less on 3d6
      | "very-bad"      // 1-3
      | "bad"           // 4-6
      | "poor"          // 7-9
      | "neutral"       // 10-12
      | "good"          // 13-15
      | "very-good"     // 16-18
      | "excellent"     // 19+
      | "unknown"       // GM hasn't decided / unrevealed
      | "secret",       // flag for relations hidden from players
  note: string,
}
```

Note: "secret" is modeling a *visibility* concept inside the type enum for
simplicity. If we later want to decouple (e.g., a Secret Alliance that's
mechanically "good"), we promote `secret` to a separate boolean field. Flagged
for future revision.

## Cascade rules

What happens when the user deletes an entity. Enforced in the delete handler.

- **Delete faction** → set `factionId = null` on all NPCs, rumors, pins, and
  quests that reference it. Remove the faction's id from every event's
  `factionIds`. Delete all relations entries that include it. Do NOT delete
  NPCs, pins, or quests themselves.
- **Delete NPC** → clear `npcSourceId` on any rumor that referenced it. Clear
  `giverNpcId` on any quest. Remove the NPC id from every event's `npcIds`.
- **Delete pin** → set `pinId = null` on NPCs and rumors and quests that
  referenced it. Remove the pin id from every event's `pinIds`.
- **Delete quest** → set `questId = null` on every event that referenced it.
  Do NOT delete events — they happened.
- **Delete event** → no cascade, it's a leaf.
- **Delete rumor** → no cascade, it's a leaf.
- **Clear a hex (terrain → unknown)** → does NOT delete pins. Pins remain
  tied to `hexKey` and stay visible. Deleting pins is always explicit.

## Derived views (computed, never stored)

- Faction → NPCs:    `npcs.filter(n => n.factionId === f.id)`
- Faction → pins:    `pins.filter(p => p.factionId === f.id)`
- Faction → rumors:  `rumors.filter(r => r.factionId === f.id)`
- Faction → quests:  `quests.filter(q => q.factionId === f.id)`
- Faction → events:  `events.filter(e => e.factionIds.includes(f.id))`
- Pin → NPCs / rumors / quests / events: as above by pin id
- Hex → pins:   `pins.filter(p => p.hexKey === key)`
- Hex → events: `events.filter(e => e.hexKeys.includes(key))`
- Quest → events: `events.filter(e => e.questId === q.id)`
- Hex → faction influence: aggregate `factionId` over the hex's pins, pick
  dominant (for optional territory-tinting layer)

## Out of scope for v1

- Timestamps on entities (createdAt, updatedAt)
- Soft delete / trash
- Change history beyond the events log
- Tags, search indexing
- Multiple campaigns in one file
- Separate "faction reputation with PCs" tracking (only faction-to-faction for now)