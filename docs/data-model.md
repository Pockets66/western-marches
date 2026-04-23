# Data Model

Single source of truth for the unified app. Any schema change must update this doc
in the same commit as the code change.

## Storage

All state lives in `localStorage` under one key: `wm_unified_v1`.
On load, the app reads that key and hydrates a single in-memory `state` object.
On every mutation, the app serializes `state` back to the same key.

The `v1` suffix is deliberate. If we ever need a breaking schema change, we write
a migration from `wm_unified_v1` to `wm_unified_v2` rather than silently corrupting
existing data.

## Top-level shape

```js
state = {
  schemaVersion: 1,
  factions: [Faction],
  npcs:     [NPC],
  rumors:   [Rumor],
  hexes:    { "col,row": Hex },   // keyed by coord string
  pins:     [Pin],
  relations: { "factionIdA|factionIdB": Relation },  // sorted key
  ui: {
    activeTab: "map" | "factions" | "rumors" | "relations",
    activeFactionId: string | null,
    activeHexKey: string | null,   // "col,row"
    rumorFilter: string,
  }
}
```

## IDs

- Every entity (faction, npc, rumor, pin) has a string `id` generated on creation.
- IDs are opaque — never parse them, never display them to the user, never rely on ordering.
- Cross-references use IDs only. Never embed a copy of one entity inside another.
- Generation: `Date.now().toString(36) + Math.random().toString(36).slice(2,6)`.

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
  interactions: [{ date: string, text: string }],  // faction-level event log
}
```

Note: NPCs are NOT nested here anymore. To get a faction's NPCs,
filter `state.npcs` by `factionId`.

### NPC

```js
{
  id: string,
  name: string,
  role: string,
  notes: string,
  factionId: string | null,    // null = independent / freelance
  pinId: string | null,        // where they're typically found, null = mobile/unknown
  log: [{ date: string, text: string }],  // player interactions
}
```

A single `factionId` and a single `pinId`. If an NPC is at multiple places
or switches factions, the event goes in `log` and the current fields are updated.
We're not modeling history as first-class data — the log is the history.

### Rumor

```js
{
  id: string,
  text: string,
  status: "unverified" | "confirmed" | "false" | "dangerous",
  playerKnown: boolean,
  factionId: string | null,    // which faction this rumor concerns
  npcSourceId: string | null,  // which NPC told the players
  // Exactly one of these two is set, or both are null:
  pinId: string | null,        // tied to a specific location
  hexKey: string | null,       // tied to a hex but no specific pin ("col,row")
}
```

Validation rule: a rumor has `pinId` set, OR `hexKey` set, OR neither. Never both.
Setting one clears the other. This is enforced in the UI, not the schema.

### Hex

Keyed by a `"col,row"` string in `state.hexes`.

```js
{
  terrain: "unknown" | "plains" | "forest" | "dark-forest" | "hills"
         | "mountain" | "water" | "swamp" | "desert" | "ruins" | "settled",
  name: string,
  note: string,
  explored: boolean,
  overlays: [Overlay],         // rivers, roads
}
```

Pins are NOT nested in hexes. To get a hex's pins, filter `state.pins` by `hexKey`.

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
  x: number,                   // local map coords, pixels within hex's local map
  y: number,
  type: "settlement" | "dungeon" | "ruin" | "landmark" | "threat" | "camp" | "other",
  name: string,
  notes: string,
  discovered: boolean,         // true = players know about it
  factionId: string | null,    // who controls this location, null = neutral/none
}
```

A pin's NPCs are derived: `state.npcs.filter(n => n.pinId === pin.id)`.
A pin's rumors are derived: `state.rumors.filter(r => r.pinId === pin.id)`.

### Relation

Keyed by `sorted([factionIdA, factionIdB]).join("|")` in `state.relations`.

```js
{
  type: "neutral" | "ally" | "friendly" | "tense" | "hostile" | "war" | "unknown" | "secret",
  note: string,
}
```

## Cascade rules when deleting

These are the rules for what happens when the user deletes an entity. Enforced
in the delete handler, not via database-style cascade.

- **Delete faction** → set `factionId = null` on all NPCs, rumors, and pins that
  reference it. Delete all relations entries that include it. Do NOT delete NPCs
  or pins themselves.
- **Delete NPC** → clear `npcSourceId` on any rumor that referenced it.
- **Delete pin** → set `pinId = null` on NPCs that lived there, clear `pinId` on
  rumors tied to it (their `hexKey` is not auto-set — the rumor becomes
  unlinked and the user can re-anchor it).
- **Delete hex / clear hex** → do NOT delete pins. Pins are tied to `hexKey`,
  and if a hex is cleared to `unknown` its pins remain. Deleting pins is always
  explicit from the local map view.
- **Delete rumor** → no cascade, it's a leaf.

## Derived views (not stored, computed on demand)

- Faction → its NPCs: `npcs.filter(n => n.factionId === f.id)`
- Faction → its pins: `pins.filter(p => p.factionId === f.id)`
- Faction → its rumors: `rumors.filter(r => r.factionId === f.id)`
- Pin → its NPCs / rumors: as above by `pinId`
- Hex → its pins: `pins.filter(p => p.hexKey === key)`
- Hex → faction influence: aggregate over hex's pins' factionIds, pick dominant
  (used for optional territory-tinting layer later)

## Out of scope for v1

- Timestamps on entities (createdAt, updatedAt)
- Soft delete / trash
- Change history beyond per-entity `log` and `interactions`
- Tags, search indexing
- Multiple campaigns in one file