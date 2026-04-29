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

## Migrations

v2 → v3:
- Add `state.players = []`, `state.sessions = []` if missing.
- Ensure every event has `sessionId: null` and `playerKnown: false` as defaults.
- Increment schemaVersion to 3.

v3 → v4:
- For each player: rename `skills` → `notes` (GM notes field); delete `level`.
- Add new fields (all `""`): `pointTotal`, `unspentPoints`, `st`, `dx`, `iq`, `ht`,
  `hp`, `will`, `per`, `fp`, `basicSpeed`, `basicMove`, `dodge`, `parry`, `block`,
  `dr`, `thrust`, `swing`, `advantages`, `disadvantages`, `skills`.
- `relations` tab removed from top-level `ui.activeTab`; any persisted value of
  `"relations"` is migrated to `"factions"` on load.
- Increment schemaVersion to 4.

v4 → v5:
- Add `state.mapMeta = { cols: 22, rows: 16, hexSize: 32 }` if missing.
- Ensure `state.hexes` is an object (default `{}`).
- Increment schemaVersion to 5.

v5 → v6:
- Replace `mapMeta.cols/rows` with signed bounds: `{ colMin: -11, colMax: 10, rowMin: -8, rowMax: 7, hexSize }`.
  The hexSize from v5 is preserved; all other fields are fixed defaults.
- Reset `state.hexes = {}` (existing hex data is incompatible with the new coordinate system).
- Set `currentHexKey = null` on every player (old keys are meaningless after the wipe).
- Increment schemaVersion to 6.

v6 → v7:
- For each rumor: add `archived: false`, `createdAt: <now>`, `archivedAt: null` if missing.
- Fix any `status: "dangerous"` → `"unverified"` (cleanup from earlier planning).
- Increment schemaVersion to 7.

v7 → v8:
- For each NPC: add `disposition: "unknown"`, `description: ""`, `secrets: ""`, `playerKnown: false` if missing.
- Increment schemaVersion to 8.

v8 → v9:
- Migration `migrateToV9()`: adds `timeUntilClock: null` and `durationClock: null` to every event.
- Increment schemaVersion to 9.

v9 → v10:
- Migration `migrateToV10()`: ensures every hex has `overlays: []` defensively.
  The `edges` constraint relaxed from "1 or 2" to "1–6" (no data change; old data with
  1–2 edges is valid in v10). New optional `surface` field added to road overlays;
  defaults to `'dirt'`. `flowFrom` semantics changed from "array index into edges"
  to "edge index 0..5 directly" — but no existing rivers had `flowFrom` set, so no
  data migration needed.
- Increment schemaVersion to 10.

v10 → v11:
- Migration `migrateToV11()`: no-op. The `Overlay.type` union gained `"route"`;
  existing overlays remain valid with no data change.
- Increment schemaVersion to 11.

v11 → v12:
- Migration `migrateToV12()`: for every hex overlay:
  - Rivers: `flowFrom: number | null` → `flowFromEdges: [number]`. If `flowFrom` was a
    number, `flowFromEdges = [flowFrom]`. If null, `flowFromEdges = []`. Delete `flowFrom`.
    Log a console warning for rivers with 2+ edges and an empty `flowFromEdges` after
    migration (audit hook for unset flow direction).
  - Rivers / roads / routes: add `trunkPair: null` if the field is absent.
  - Water overlays (`type: "water"`) are new in v12; no migration needed.
- Increment schemaVersion to 12.

v13 → v14 (slice 15 — pins + local maps):
- Migration `migrateToV14()`: add `state.localMaps = {}` if missing; add `state.pins = []` if
  missing; for each pin ensure all fields present (`discovered`, `factionId`, `notes`, `type`,
  `x`, `y`, `hexKey`, `name`).
- Increment schemaVersion to 14.

v12 → v13 (slice 14.10 — segment model consolidation):
- Migration `migrateToV13()`: for every hex overlay:
  - Water: add `segments: []`, delete `edges`.
  - River / road / route: build `segments` from legacy `edges`/`flowFromEdges` fields via
    `legacyEdgesToSegments()`, then delete `edges`, `flowFromEdges`, `trunkPair`,
    `originAtCenter`. If `segments` already present, leave it.
  - `legacyEdgesToSegments` mapping: 1 edge → `{C↔edge}`; 2 edges (no centerOrigin) →
    `{edgeA↔edgeB}` with flow derived from inflows; 3+ edges or centerOrigin → fan
    `{C↔each}`.
- This migration is belt-and-suspenders only — no real user data existed at v12.
- Increment schemaVersion to 13.

**Consolidation history (slice 14.10):** Replaced the edge-based overlay model (slices
14–14.9 exploratory phase) with a segment-based model. Slice 14.9 (center-origin support)
and slice 14.6 (map undo) are deferred to slice 14.11. The edge/flowFromEdges/trunkPair
fields are removed entirely from the schema.

Incoming data from partner files (one-time import, not automatic):
- `wm_sessions_v1` → read players into `state.players`, read sessions into
  `state.sessions` (map the existing `planned`/`actual` fields, initialize
  narrative-layer fields to empty).
- `wm_explog_v1` → read expedition entries and merge them into matching sessions
  by date/title where possible, or create new sessions with null `hooks`/`planned`
  and populated narrative-layer fields.

Import should be a deliberate user action, not automatic — see future slice.

## Top-level shape

```js
state = {
  schemaVersion: 14,
  factions:    [Faction],
  npcs:        [NPC],
  rumors:      [Rumor],
  quests:      [Quest],
  events:      [Event],
  players:     [Player],
  sessions:    [Session],
  mapMeta:     MapMeta,
  hexes:       { "col,row": Hex },
  pins:        [Pin],
  localMaps:   { "col,row": LocalMap },
  relations:   { "factionIdA|factionIdB": Relation },
  ui: {
    activeTab: "map" | "factions" | "rumors" | "quests"
             | "sessions" | "players" | "npcs" | "events",
    activeFactionId: string | null,
    activeQuestId:   string | null,
    activeSessionId: string | null,
    activePlayerId:  string | null,
    activeNpcId:     string | null,
    activeEventId:   string | null,
    activeHexKey:    string | null,
    rumorFilter:          string,
    questFilter:          string,
    sessionFilter:        string,
    eventFilter:          string,
    eventFactionFilter:   string,
    eventUrgencyFilter:   string,
    eventPlayerKnownOnly: boolean,
    mapMode:        string | null,   // current map tool mode
    rulerHexA:      string | null,   // "col,row" of first picked ruler hex
    rulerHexB:      string | null,   // "col,row" of second picked ruler hex
    rulerMove:      number,          // party Move score (default 5)
    rulerOverrides: { [hexKey]: string | null },  // per-hex modifier overrides; clears on panel close
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

### Player
```js
{
  id: string,
  name: string,               // the real human's name (or handle)
  character: string,          // the PC's in-game name
  cls: string,                // profession / template — free text (UI label: "Profession / Template")
  currentHexKey: string|null, // "col,row" — validated against state.hexes; invalid keys flagged in UI
  color: string,              // hex color for sidebar dot and card accent
  prefs: string,              // playstyle notes, favourite tactics

  // GM-facing notes (was `skills` in v3; renamed to avoid collision with GURPS skills)
  notes: string,

  // Point accounting
  pointTotal:    string,      // total character points (freeform)
  unspentPoints: string,      // unspent points remaining

  // Primary attributes (GURPS)
  st: string, dx: string, iq: string, ht: string,

  // Secondary characteristics
  hp: string,         // default = ST
  will: string,       // default = IQ
  per: string,        // default = IQ
  fp: string,         // default = HT
  basicSpeed: string, // default = (HT+DX)/4
  basicMove: string,  // default = floor(basicSpeed)

  // Defenses & damage
  dodge: string, parry: string, block: string, dr: string,
  thrust: string, swing: string,

  // Free-text lists (one entry per line)
  advantages:    string,
  disadvantages: string,
  skills:        string,

  loot: [string],   // freeform list of items/notes
}
```

All numeric fields are stored as strings. No auto-computation. Tier 2+ will add
derived values and point-cost breakdowns.

### Faction

```js
{
  id: string,
  name: string,
  color: string,               // hex, e.g. "#c8972a"
  alignment: {
    ethical: "lawful" | "neutral" | "chaotic" | null,
    moral:   "good"   | "neutral" | "evil"    | null,
  },
  activity: "dormant" | "active" | "disbanded",
  size: "tiny" | "small" | "medium" | "large" | "giant" | "unknown",
  // size thresholds: tiny (<5), small (6–20), medium (21–100), large (101–1000), giant (1000+)
  goals: string,
  notes: string,
  playerKnown: boolean,
  clocks: [
    {
      id: string,
      size: 4 | 6 | 8 | 10 | 12,
      filled: number,          // 0..size
      label: string,
      consequence: string,
    }
  ],
}
```

Faction-level session history is no longer stored here. To get a faction's
event history, filter `state.events` by `factionIds`.

**Migration note (v1):** Old data may have `threat` and a single `clock` object.
`migrateFactions()` in `load()` handles the upgrade automatically:
- `threat` → `activity` (dormant→dormant, rising/active→active, defeated→disbanded)
- `clock` → `clocks: [{ id, ...clock }]`
- Adds `alignment: { ethical: null, moral: null }`, `size: "unknown"`

### NPC

```js
{
  id: string,
  name: string,
  role: string,
  notes: string,
  factionId:   string | null,  // null = independent / freelance
  pinId:       string | null,  // where they're typically found; null = mobile/unknown
  disposition: "friendly" | "neutral" | "hostile" | "unknown",  // default "unknown"
  description: string,         // physical appearance, voice, mannerisms
  secrets:     string,         // GM-only: what does this NPC know?
  playerKnown: boolean,        // default false — have players encountered this NPC?
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
  archived:   boolean,          // true = moved to archive tab
  createdAt:  string,           // ISO timestamp
  archivedAt: string | null,    // ISO timestamp when archived, else null
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

A thing that happens NOT during a session, linkable to any combination of entities.

```js
{
  id: string,
  date: string,                // "Session 12", "1/15", freeform
  text: string,
  urgency: Urgency,
  playerKnown: boolean,        // has this been revealed to players?
  timeUntilClock: { size: 4|6|8|10|12, filled: number } | null,
  durationClock:  { size: 4|6|8|10|12, filled: number } | null,

  // Links — any combination, all optional
  factionIds:  [string],
  npcIds:      [string],
  pinIds:      [string],
  hexKeys:     [string],
  questId:     string | null,
  sessionId:   string | null,  // if this happened during a session
}
```

Each clock is optional (`null` = empty slot). The slot itself identifies the
clock — there are no `id`, `label`, or `consequence` fields as faction clocks
have. `timeUntilClock` counts down to when the event triggers; `durationClock`
tracks how long the event lasts once it starts. `filled` is clamped to `[0, size]`.

Events replace the `interactions` array previously on factions and the `log`
array previously on NPCs. "Faction X's history" is now a derived filter, not
stored data.

### Session

```js
{
  id: string,
  title: string,              // "Session 12" or a thematic title
  date: string,               // freeform — "Jan 15", "2026-01-15", whatever
  playerVisible: boolean,     // can the in-world report be shown to players?
  
  // Attendance: which players were present
  attendance: { [playerId]: boolean },
  
  // PLANNING LAYER — filled in before the session
  hooks: [{ text: string, source: string }],  // active hooks going in
  planned: string,                             // what the GM prepared
  
  // PLAY LAYER — filled in during/after the session
  actual: string,                              // quick actual-play notes (GM-facing)
  
  // NARRATIVE LAYER — the expedition-log style after-action record
  location: string,                            // region / starting point — freeform
  report: string,                              // in-world narrative, PC-perspective
  gmNotes: string,                             // hidden GM analysis
  
  // TAGGED LISTS — optional free-text with autocomplete against real records
  hexesVisited: [string],                      // each item is a hexKey OR free text
  factionsEncountered: [{ text: string, sub: string }],  // text = name, sub = NPC name
  loot:       [{ text: string, sub: string }],
  casualties: [{ text: string, sub: string }],
  misc:       [{ text: string, sub: string }],
}
```

### MapMeta

Grid configuration. Stored at `state.mapMeta` (not in any collection).

```js
{
  colMin:  number,   // inclusive signed integer (default -11)
  colMax:  number,   // inclusive signed integer (default  10)
  rowMin:  number,   // inclusive signed integer (default  -8)
  rowMax:  number,   // inclusive signed integer (default   7)
  hexSize: number,   // hex circumradius in pixels (default 32, min 16, max 64)
}
```

Hex `(0, 0)` is the world origin. Defaults give a 22×16 grid centered on `(0, 0)`.
The map can be expanded asymmetrically in any direction.

Canvas pixel dimensions are derived using pointy-top hex geometry:
- `cols = colMax - colMin + 1`
- `rows = rowMax - rowMin + 1`
- `HW = sqrt(3) * hexSize`  (hex flat width)
- `VS = hexSize * 1.5`      (vertical step)
- `W  = ceil(cols * HW + HW/2 + PAD*2)`
- `H  = ceil(rows * VS + hexSize*0.5 + PAD*2)`

### Hex

Keyed by `"col,row"` in `state.hexes`. Both `col` and `row` are signed integers,
so keys like `"-5,3"` or `"0,-2"` are valid and expected.

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

Anchor points: `0=E, 1=SE, 2=SW, 3=W, 4=NW, 5=NE` (edge midpoints) plus `'C'`
(hex center). Edge indices derived from `hexCorner` angle formula `60*i − 30` degrees.

```js
// Segment (shared shape for river/road/route)
{
  from: 'C' | 0 | 1 | 2 | 3 | 4 | 5,   // anchor — 'C' is center, integers are edge midpoints
  to:   'C' | 0 | 1 | 2 | 3 | 4 | 5,   // anchor — must differ from `from`
  flow: 'fromTo' | 'toFrom' | null       // rivers only; ignored on roads/routes
}
// Constraints: from !== to; no two segments in one overlay share the same {from,to} pair.

// River
{ type: "river", segments: [Segment] }

// Road
{ type: "road", surface: "dirt" | "gravel" | "stone", segments: [Segment] }

// Route
{ type: "route", segments: [Segment] }

// Water
{ type: "water", size: "pond" | "lake", segments: [] }  // segments always empty
```

**Segment semantics.** A segment is a connection between any two of a hex's 7 anchor
points. Multiple segments in one overlay are independent — they need not connect. This
naturally expresses:
- `{C, edge}` — stub spoke from center to an edge
- `{edge1, edge2}` — smooth pass-through (no center waypoint → no center dip)
- Multiple `{C, edgeN}` segments — junction at center (Y-shape, star)
- Two disconnected `{edgeA, edgeB}` segments — parallel non-connecting roads

**Flow semantics (rivers only).** `flow: 'fromTo'` means water flows from the `from`
anchor to the `to` anchor. `flow: 'toFrom'` means the reverse. `flow: null` means no
direction is set; no flow arrow at that segment's terminal. Flow arrows render only at
network terminal nodes (degree-1 nodes with no cross-hex continuation).

**Water overlays** are independent of terrain and of other overlay types. A hex may
have at most one water overlay. Water overlays do not participate in the network
graph. A hex can have a water overlay AND a river overlay simultaneously; river
splines clip to the water bounding circle (pond: 25% apothem radius; lake: 50%).

**Route overlays** are visually distinct (red dashed, `#a02828`). Created by the
travel calculator's "Mark route on map"; removed via Erase Routes toolbar. Routes do
not grant the road speed bonus in travel calculations (only `type === "road"` does).

**Rendering (slice 14.10+).** Segments are assembled into a per-type graph. Connected
runs are extracted (maximal paths from terminal/junction to terminal/junction) and
rendered as centripetal Catmull-Rom splines (alpha=0.5, Barry-Goldman, parameterized
by sqrt(distance)) through anchor offset positions. Consecutive duplicate waypoints
are deduped before splining; interior C nodes with graph degree 2 are dropped at
render time so accidental center dips render smooth. A 2-segment pass-through
`{e1, e2}` (no center anchor) produces a smooth curve that never dips to the hex
center. At junctions (degree-3+ nodes in the graph), the two
most-opposite-tangent runs are merged into a continuous trunk spline; remaining runs
peel off as branches. Per-type perpendicular offsets (river −3px, road +3px, route
+6px) keep overlays visually separated on shared edges. Render z-order: terrain →
rivers → ponds → roads → lakes → routes → path-highlight. Lakes render on top of
roads, visually blocking any road segment that passes through the hex center.
`modifierFor()` mirrors this: a lake + center-touching road segment gets lake terrain
rules, not road bonus.

**Lake-blocks-roads rule.** A hex with a lake overlay AND road segments that have
`'C'` as an endpoint: road bonus suppressed (lake blocks the road). A hex with a lake
AND only edge-to-edge road segments (no center): road bonus applies (road goes around).
Ponds never block roads.

**Cascade.** Overlays are leaves attached to hexes — no other entity references
them. Clearing a hex's terrain does NOT clear its overlays (roads and rivers are
independent of terrain). Resizing the map to shrink bounds deletes out-of-bounds
hexes entirely, which removes their overlays as part of the hex deletion.

### LocalMap

Stored at `state.localMaps["col,row"]` — one entry per hex, optional.

```js
{
  imageDataUrl: string | null,   // base64 JPEG data URL (≤1600px longest dim, quality 0.85), or null
  width:  number,                // canvas width in pixels
  height: number,                // canvas height in pixels
}
```

Default for a blank local map: `{ imageDataUrl: null, width: 800, height: 600 }`.
When `imageDataUrl` is null the canvas renders a parchment-tinted background.

### Map Undo

In-memory only — not persisted to `localStorage` and clears on every page reload.

- **Scope:** `state.mapMeta` + `state.hexes` + `state.pins` + `state.localMaps`. Other tabs unaffected.
- **Stack cap:** 50 entries per stack (undo and redo); oldest entry dropped when exceeded.
- **Snapshot model:** each entry is `JSON.stringify({ mapMeta, hexes, pins, localMaps })`. Restoring
  parses it, assigns all four fields, then calls `save()` — so `localStorage` tracks the
  post-undo state. If the user reloads, they see the post-undo state with an empty stack.
- **Drag = one unit:** a transaction captures a single pre-drag snapshot at mousedown
  for overlay paint/erase and pin drag; committed on mouseup/mouseleave.
- **Import clears stacks:** after a successful map import both stacks are reset (prior
  history is irrelevant to a different map).

### Pin

```js
{
  id: string,
  hexKey: string,              // "col,row", which hex it lives in
  x: number,                   // local map coords, pixels (default 400)
  y: number,                   // local map coords, pixels (default 300)
  type: "settlement" | "dungeon" | "ruin" | "landmark" | "threat" | "camp" | "other",
  name: string,
  notes: string,
  discovered: boolean,         // true = players know about it
  factionId: string | null,    // who controls this location
}
```

**World map rendering.** Pins render as small colored dots (radius `hexSize * 0.12`) clustered
around the hex center. Color is the faction color if `factionId` is set, else a per-type default.
Undiscovered pins use a dashed stroke. Pins on `terrain: 'unknown'` hexes are hidden.

**Local map rendering.** Pins render at their `(x, y)` coordinates as 14px circles with a glyph
and name label below. The selected pin gets a gold highlight ring. Pins are draggable via the
local map modal.

**Cascade rules:**
- **Delete faction** → set `factionId = null` on all pins referencing it. Pin dots recolor to
  type-default color.
- **Delete pin** → set `pinId = null` on referencing NPCs, rumors, quests; remove from
  `event.pinIds` arrays.
- **Resize map (shrink)** → pins whose `hexKey` falls outside the new bounds are deleted;
  `state.localMaps[hexKey]` entries for deleted hexes are also removed.
- **Clear hex terrain** (erase tool) → does NOT delete pins or local map.

**Cross-links** (UI available from slice 17/18 onwards):
A pin's NPCs:    `state.npcs.filter(n => n.pinId === pin.id)`
A pin's rumors:  `state.rumors.filter(r => r.pinId === pin.id)`
A pin's quests:  `state.quests.filter(q => q.pinId === pin.id)`
A pin's events:  `state.events.filter(e => e.pinIds.includes(pin.id))`

### Overlay

*(See the Overlay section under Hex above.)*

### Relation

Keyed by `sorted([factionIdA, factionIdB]).join("|")` in `state.relations`.

```js
{
  type: "none"           // factions have never met / no relationship exists
      | "unknown"        // relationship exists, GM hasn't decided the posture yet
      | "mortal-enemies" // 3d6: 0 or less equivalent
      | "sworn-enemies"  // 1-3
      | "hostile"        // 4-6
      | "tense"          // 7-9
      | "neutral"        // 10-12
      | "friendly"       // 13-15
      | "allied"         // 16-18
      | "sworn-allies",  // 19+
  playersAware: boolean, // true = players know the true relationship
  note: string,
}
```

Vocabulary note: the type values describe *long-term posture*, not an
instantaneous reaction roll. The 3d6 equivalences are listed so the GM can
translate quickly from GURPS reaction mechanics, but the semantics are
ongoing political stance between two factions.

Special states `none` and `unknown` are distinct:
- **none**: no relationship exists (factions haven't interacted / don't know
  each other).
- **unknown**: a relationship exists but the GM hasn't decided the posture yet.
  Use this as the default for new relations the GM hasn't touched.

### Relation UI rules

- When the user attempts to change the `type` on a relation where
  `playersAware === true`, show a confirmation prompt of the form:
  "The players know this relationship as *<current type>*. Changing it to
  *<new type>* represents a narrative event (betrayal, alliance, reveal).
  Proceed?"
  The prompt must be dismissible. No confirmation is needed if
  `playersAware === false`.
- Toggling `playersAware` itself never prompts.
- Changing `note` never prompts.

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
- **Clear a hex (terrain → unknown)** → does NOT delete pins or overlays. Roads
  and rivers are independent of terrain and remain intact. Deleting pins and
  removing overlays are always explicit operations.
- **Delete player** → remove the player id from every session's `attendance` object.
  Do NOT delete sessions.
- **Delete session** → set `sessionId = null` on every event that referenced it.
  Do NOT delete events — they happened.
- **Resize map (shrink bounds)** → hexes that fall outside the new bounds are
  deleted via `delete state.hexes[k]`, removing their overlays as part of the
  hex. No other cascade needed.

## Derived views (computed, never stored)

- Faction → NPCs:    `npcs.filter(n => n.factionId === f.id)`
- Faction → pins:    `pins.filter(p => p.factionId === f.id)`
- Faction → rumors:  `rumors.filter(r => r.factionId === f.id)`
- Faction → quests:  `quests.filter(q => q.factionId === f.id)`
- Faction → events:  `events.filter(e => e.factionIds.includes(f.id))`
- NPC → events:      `events.filter(e => e.npcIds.includes(n.id))`
- Pin → NPCs / rumors / quests: as above by pin id
- Pin → events:      `events.filter(e => e.pinIds.includes(p.id))`
- Hex → pins:   `pins.filter(p => p.hexKey === key)`
- Hex → events: `events.filter(e => e.hexKeys.includes(key))`
- Quest → events: `events.filter(e => e.questId === q.id)`
- Session → events:    `events.filter(e => e.sessionId === s.id)`
- Hex → faction influence: aggregate `factionId` over the hex's pins, pick
  dominant (for optional territory-tinting layer)
- Player → sessions:   `sessions.filter(s => s.attendance[p.id])`
- Hex → players:       `players.filter(pl => pl.currentHexKey === key)`
- Faction → sessions:  all sessions where this faction appears in `factionsEncountered`
  (substring match on `text`, since it's free text for now)

## Out of scope for v1

- Timestamps on entities (createdAt, updatedAt)
- Soft delete / trash
- Change history beyond the events log
- Tags, search indexing
- Multiple campaigns in one file
- Multiple characters per player (collapsed for now)
- Per-PC character sheets (only free-text fields)
- Automatic link resolution on session tagged lists (stays free text)
- Import from partner files (deferred to its own slice)
