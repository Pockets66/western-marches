# Slice 20.5 — File-backed campaign storage

Replaces `localStorage` as the source of truth with real `.json` files on disk,
opened and autosaved through the File System Access API. Adds a launch screen
with a recents list so multiple campaigns can coexist and be switched between.

**No schema change.** The serialized `state` shape is byte-identical to v15;
only its destination changes. This slice is schema-neutral and can be
resequenced freely relative to slices 20 and 21.

> **Numbering note.** Filed as 20.5 on the assumption slice 20 (pathfinder)
> runs first. If you'd rather land storage before the pathfinder, renumber to
> 19.5 — nothing in here depends on slice 20.

## Motivation

Three problems, one cause:

1. **The 5MB ceiling is close.** Slice 15 stores local map images as base64
   JPEG data URLs inside the single `wm_unified_v1` key. `localMapUploadImage`
   already has a "Storage full — image not saved." rollback path, which means
   the failure mode was anticipated at build time. Every local map added moves
   the campaign nearer that wall.
2. **One browser profile, one campaign.** A single storage key means a second
   campaign would overwrite the first. Running two games concurrently is
   currently impossible without manual export/import juggling.
3. **Backups are a ritual, not a property.** The working agreement "localStorage
   backup before any schema-changing slice" exists because the data lives
   somewhere fragile. Clearing site data destroys a campaign.

File-backed storage dissolves all three: no practical size limit, one file per
campaign, and backups become "copy the file" or `git add`.

It also creates the save/load seam the eventual Firestore migration will plug
into, so this work is not thrown away later.

## Design decisions

- **`save()` keeps its synchronous signature.** It is called from ~everywhere
  (every `update*Field`, `_mapRestoreSnapshot`, `switchTab`, all CRUD). It
  becomes: mark dirty, schedule a debounced flush, return. Call sites are
  untouched. This is the single most important constraint in the slice — do not
  convert `save()` to async.
- **Debounce at 800ms, with a hard 5s ceiling.** A drag-paint across 12 hexes
  produces one write, not twelve. The ceiling guarantees that continuous typing
  in a notes textarea still flushes at least every 5 seconds.
- **`saveNow()` exists for the few places that must block.** Returns a promise
  that resolves after a flush. Used by `localMapUploadImage` (see below), and
  before closing/switching campaigns.
- **localStorage becomes a best-effort crash mirror, not the source of truth.**
  Every flush also attempts a localStorage write inside `try/catch`; failure is
  silent and non-blocking. Its only job is recovering work if the browser dies
  between flushes. Because failure is tolerated, the 5MB ceiling stops being a
  functional limit — the file write is what matters.
  > **Confirm:** the alternative is dropping localStorage entirely, which is
  > purer w.r.t. "no browser storage" but loses crash recovery. Recommending
  > keep-as-mirror. Say the word and stage 1 drops it instead.
- **Handles are persisted in IndexedDB; campaign data is not.** A
  `FileSystemFileHandle` cannot be JSON-serialized, so the recents list requires
  IndexedDB (store `wm_handles`, one record per recent campaign holding the
  handle, display name, and last-opened timestamp). Campaign *content* never
  touches IndexedDB. Wiping it costs the convenience list only.
- **Campaign name is derived from the filename.** `frostmarch.json` displays as
  "Frostmarch". No `state.meta.name` field, no schema bump.
  > **Confirm:** a `state.meta = { name, createdAt }` field would allow renaming
  > independent of the file, at the cost of a v16 bump. Recommending filename-
  > derived for simplicity.
- **Permission re-prompt is accepted, not fought.** Chrome requires
  `queryPermission`/`requestPermission` on a stored handle once per session. The
  launch screen makes this a single click and does not attempt to hide it.
- **No lock against opening the same file twice.** Two tabs on one campaign is a
  last-write-wins footgun. Out of scope; noted in the docs.

## Stages

Stop after each stage for explicit confirmation. One commit per stage.

---

### Stage 1 — Storage seam (no behavior change)

Pure refactor. At end of stage the app looks and behaves exactly as it does
today, still reading and writing localStorage — but through an abstraction with
dirty-tracking and debounced flushing already in place.

Introduce near the existing `STORAGE_KEY` declaration:

```js
const STORAGE_KEY = 'wm_unified_v1';   // retained: crash mirror only
let   _storeDirty     = false;
let   _storeFlushTimer = null;
let   _storeFirstDirty = 0;
const _STORE_DEBOUNCE_MS = 800;
const _STORE_MAX_WAIT_MS = 5000;
let   _storeBackend   = 'local';       // 'local' | 'file' — 'file' lands stage 2
```

- Rewrite `save()` to: set `_storeDirty = true`; record `_storeFirstDirty` if
  this is the first dirty mark; schedule/reschedule the flush timer, clamped so
  the flush never sits longer than `_STORE_MAX_WAIT_MS` past `_storeFirstDirty`.
- Add `_flush()` — serializes `state` once, dispatches to the active backend,
  clears the dirty flag. For now the only backend is the localStorage write.
- Add `saveNow()` → cancels the pending timer, runs `_flush()`, returns a
  promise.
- Add a `beforeunload` handler that flushes synchronously if dirty.
- Add a save-status element to the header (right-aligned, muted, Cinzel small
  caps): idle shows the campaign name, dirty shows "Saving…", settled shows
  "Saved ✓" fading to idle after ~1.5s. Reuse the existing toast/flash styling
  vocabulary; new CSS class prefix `st-*`.

**Rework `localMapUploadImage`.** Its current `try { save(); } catch { rollback }`
will no longer catch anything once writes are deferred. Change it to `await
saveNow()` inside a `try/catch`, preserving the existing rollback (restore
`prevDataUrl`/`prevW`/`prevH`, pop `_mapUndoStack`, `_refreshUndoButtons()`,
redraw, toast). Make the enclosing `img.onload` handler `async`. This is the one
call site that genuinely needs a blocking save; do not convert others.

**Acceptance criteria**
- App loads, all tabs render, all existing behavior unchanged.
- Typing 20 characters into a hex note produces 1–2 localStorage writes, not 20
  (verify by wrapping `localStorage.setItem` in a counting shim in DevTools).
- Drag-painting across 8 hexes produces one write.
- Uploading a local map image still rolls back correctly when quota is exceeded
  (test by first filling storage with a large image).
- Reloading mid-edit loses nothing: `beforeunload` flushed.
- Map undo/redo still round-trips (it calls `save()` via `_mapRestoreSnapshot`).

---

### Stage 2 — File backend and handle persistence

The file layer, driven from a temporary debug button. No launch screen yet.

- **Feature detect.** `_fsSupported()` → `'showOpenFilePicker' in window`. If
  false, the app stays on the localStorage backend permanently and the launch
  screen (stage 3) is skipped entirely. Never leave the user with a dead app.
- **IndexedDB handle store.** Database `wm_store`, object store `wm_handles`,
  keyed by an opaque id. Record shape:
  `{ id, handle, displayName, fileName, lastOpened }`. Thin promise wrappers:
  `_idbPut`, `_idbGetAll`, `_idbDelete`. No library.
- **`openCampaign()`** — `showOpenFilePicker` with a JSON file-type filter, read
  the file, `JSON.parse`, run the *existing* migration chain (`migrateToV*`,
  identical to what `load()` does today — factor that chain into a shared
  `_hydrate(parsed)` so file and localStorage paths cannot drift), assign
  `state`, set `_storeBackend = 'file'`, upsert the handle record, full re-render.
- **`newCampaign()`** — `showSaveFilePicker`, default name `campaign.json`,
  write a fresh default state, then proceed as `openCampaign`.
- **`_verifyPermission(handle)`** — `queryPermission({mode:'readwrite'})`, and if
  not `'granted'`, `requestPermission`. Returns boolean.
- **File flush path** — `_flush()` when `_storeBackend === 'file'`:
  `createWritable()`, write, `close()`. Wrap in try/catch; on failure show a
  persistent (non-auto-dismissing) error banner reading "Could not write to
  <file> — your changes are unsaved." with a "Reconnect" button that re-runs
  `_verifyPermission`. Silent write failure is the worst possible outcome here;
  make it loud.
- Temporary toolbar buttons "Open" / "New" to drive this manually. Removed in
  stage 3.

**Acceptance criteria**
- Open a `.json` written by hand containing a v15 state → loads correctly.
- Open a *v8-era* export → migration chain runs, upgrades to v15, loads.
- Edit a hex, wait 1s, open the file in Notepad → change is present.
- Revoke permission via the site-settings padlock mid-session → error banner
  appears, Reconnect restores writing.
- `_fsSupported()` forced false → app behaves exactly as it does today.

---

### Stage 3 — Launch screen and recents

Replace the temporary buttons with the real entry experience.

- **Boot sequence change.** The current init block is bare synchronous:
  `load(); renderFactionSidebar(); switchTab(...)`. Wrap it in an `async function
  boot()`. `boot()` decides: no recents and no legacy localStorage → show the
  launch screen empty state; otherwise show launch screen with recents.
- **Launch screen** — full-viewport overlay, parchment/gold styling consistent
  with the app, Cinzel heading "Western Marches Tracker". Contains:
  - Recents list, newest first, each row: display name, filename in muted text,
    relative last-opened ("2 days ago"). Click → verify permission → load.
  - "Open campaign…" and "New campaign…" buttons.
  - Per-row "✕" to forget a recent (removes the IndexedDB record only — never
    touches the file on disk; make that explicit in the confirm text).
  - If a stored handle's file has been moved or deleted, the row loads-fails
    gracefully: toast "Couldn't find <file>", offer to forget the entry.
- **Legacy import.** If `localStorage[STORAGE_KEY]` exists and holds a real
  campaign, show a distinct card at the top: "Import existing campaign from
  browser storage". Clicking runs `showSaveFilePicker`, writes the migrated
  state to the chosen file, and opens it. **Do not delete the localStorage copy**
  — leave it untouched as a safety net for at least this slice.
- CSS prefix for this screen: `lc-*`.

**Acceptance criteria**
- Fresh profile (no IndexedDB, no localStorage) → empty launch screen, New works.
- Existing localStorage campaign → import card appears, import produces a file
  whose contents match the previous state exactly (diff the JSON).
- After import, localStorage key still present.
- Recents survive a browser restart; permission prompt appears once per session.
- Deleting a campaign file outside the browser, then clicking its recent row →
  graceful failure, no console errors, offer to forget.

---

### Stage 4 — Switching campaigns and polish

- **"Switch campaign" control** in the header, next to the save status.
  `await saveNow()` → tear down → return to launch screen. Must be safe mid-edit.
- **Full state reset on switch.** This is the highest-risk item in the slice.
  Audit and reset every module-local that outlives `state`, at minimum:
  `_mapUndoStack`, `_mapRedoStack`, `_mapUndoTxnSnap`, `_mapSelHex`,
  `_localmapHexKey`, `_localmapImgCache`, `_lmSelPinId`, `_lmMode`, plus the
  overlay and pin caches (`_invalidateOverlayCache()`, `_invalidatePinCache()`).
  Grep for `^let _` in the script block and account for every one. A leaked
  `_localmapImgCache` across campaigns means one campaign's map art rendering on
  another's hexes.
- **Window title** reflects the open campaign: `Frostmarch — Western Marches
  Tracker`. Makes two windows on two campaigns distinguishable in the taskbar.
- **Documentation.**
  - `CLAUDE.md`: rewrite the persistence constraint. It currently reads
    "Persistence is `localStorage` under a single key: `wm_unified_v1`" — replace
    with the file-backed model, noting localStorage's demotion to crash mirror
    and IndexedDB's handle-only role. Also correct the stale "Schema is at v8"
    line to v15 while you're in there.
  - `docs/data-model.md`: rewrite the "## Storage" section wholesale. Document
    the file format (identical to the old serialized state), the debounce
    semantics, the crash-mirror role, the IndexedDB handle store, and the
    no-multi-tab-lock caveat.
  - `ROADMAP.md`: add slice 20.5 to Done. Also correct the stale in-progress
    section — it still shows slice 18 in progress and slice 19 as the narrative-
    event recorder, which was dropped.

**Acceptance criteria**
- Open campaign A, paint hexes, upload a local map image, undo twice, switch to
  campaign B: no A data visible anywhere in B, undo stack empty, no A imagery.
- Switch back to A: everything intact including the local map image.
- Two browser windows, two different campaigns, both editing → no interference.
- Title bar shows the right name in both.

## Non-goals

- Any schema change. If one becomes necessary, stop and re-plan.
- Multi-tab locking or conflict detection on the same file.
- Cloud sync, Firestore, or anything player-facing.
- Auto-save-to-cloud, versioning, or file history beyond what git gives you.
- Changing `exportMap`/`importMap` — those remain the map-only JSON subset and
  are unaffected by this slice.
- Removing localStorage entirely (deferred; see the confirm flag in stage 1).
- Encrypting or compressing the campaign file.

## Schema notes

Schema stays at **v15**. The migration chain is *reused unchanged* by the file
loader — stage 2 factors it into `_hydrate(parsed)` shared by both paths
specifically so a future schema bump only has to be written once. Verify after
stage 2 that a v8 export still upgrades correctly through the file path.

## Smoke test plan

Run end-to-end after stage 4, from a fresh browser profile:

1. Launch → empty screen → New campaign → `test-a.json`.
2. Paint terrain, add a road, add a pin, upload a local map image, write notes.
3. Confirm "Saved ✓" settles. Open `test-a.json` in Notepad — content present.
4. Close the tab without warning. Reopen `index.html` → recents shows Test A →
   click → permission → everything restored.
5. New campaign → `test-b.json`. Verify Test A's map, pins, image are absent.
6. Switch back to Test A. Verify complete.
7. Copy `test-a.json` to `test-a-backup.json` in Explorer. Open the backup as a
   third campaign. Verify it loads as an independent copy.
8. Kill the browser process mid-edit (Task Manager, within the debounce window).
   Reopen → confirm the crash mirror or last flush recovered the work, and that
   whichever it was, the app said so clearly rather than silently losing an edit.
9. Confirm the app still works with no network connection at all.

## Workflow

- **Back up localStorage before starting**, per the standing agreement — even
  though this slice doesn't bump schema, it's the slice most likely to touch the
  campaign's only copy.
- Also `git tag pre-file-storage` before stage 1. This is an architectural change
  and a clean rollback point is worth the two seconds.
- One commit per stage: `slice 20.5 stage N — <summary>`.
- Commit this prompt file to `prompts/slice-20.5.md` before stage 1 begins.
- PowerShell 5.1: no `&&`. Separate lines for every git command.
- Doc updates (`CLAUDE.md`, `docs/data-model.md`, `ROADMAP.md`) land in the
  stage 4 commit.

## Definition of done

- Campaigns live in files chosen by the user; the app never depends on
  localStorage for correctness.
- Multiple campaigns coexist and switch cleanly with no state bleed.
- Autosave is invisible in normal use and loud on failure.
- The app still opens from `file://` with no network and no build step.
- `save()` still has the same signature it had at the start of the slice.
