Slice 5: refactor the Faction schema — replace threat with alignment/activity/size axes, and replace single clock with a clocks array.
Schema changes in docs/data-model.md:
Update the Faction entity to:

js{
  id: string,
  name: string,
  color: string,
  alignment: {
    ethical: "lawful" | "neutral" | "chaotic" | null,
    moral:   "good"   | "neutral" | "evil"    | null,
  },
  activity: "dormant" | "active" | "disbanded",
  size: "tiny" | "small" | "medium" | "large" | "giant" | "unknown",
  goals: string,
  notes: string,
  playerKnown: boolean,
  clocks: [
    {
      id: string,
      size: 4 | 6 | 8 | 10 | 12,
      filled: number,
      label: string,
      consequence: string,
    }
  ],
}

Note the comment thresholds for size: tiny (<5), small (6-20), medium (21-100), large (101-1000), giant (1000+), unknown (undecided).
Remove the old threat field and the old single clock field from the documented schema entirely.
Migration:
Existing data in wm_unified_v1 has threat and clock on each faction. Write a migration function that runs on load() when it detects the old shape. Migration rules:

If a faction has threat but no activity, map it: dormant → dormant, rising → active, active → active, defeated → disbanded. Then delete the threat field.
If a faction has clock (singular) but no clocks (plural), wrap it: clocks = [{ ...existingClock, id: uid() }]. Then delete clock.
If a faction has no alignment, set it to { ethical: null, moral: null }.
If a faction has no size, set it to "unknown".

After migration, increment schemaVersion to 2 in state and in the doc. The storage key stays wm_unified_v1 (we're not breaking the key, just migrating the shape).
UI changes in the Factions panel:
Remove the threat pills row. Add three new controls in its place:

Alignment picker. Two dropdowns side by side: Ethical (Lawful / Neutral / Chaotic / —) and Moral (Good / Neutral / Evil / —). The "—" option sets the axis to null.
Activity pills. Three-option pill selector: Dormant / Active / Disbanded.
Size dropdown. Six options with the threshold hints in parentheses as helper text: "Tiny (<5)", "Small (6-20)", "Medium (21-100)", "Large (101-1000)", "Giant (1000+)", "Unknown".

Update the sidebar faction list: the threat badge becomes a compact activity indicator (e.g. a colored dot: grey for dormant, amber for active, faded/strikethrough for disbanded). Don't try to cram alignment or size into the sidebar — they live in the detail view only.
Replace the single Agenda Clock section with a Clocks section:

Each clock renders as it did before (SVG with label input, segment picker, advance/remove buttons, consequence textarea).
An "+ Add Clock" button at the top of the section adds a new empty clock to the array.
Each clock has a delete button (confirm before removing).
Clocks display in the order they're stored in the array — no automatic sorting.

Check that Relations still works:
After the refactor, the Relations tab should still render correctly. Relations only reads faction.name, faction.color, and faction.id, so it shouldn't be affected — but verify and flag anything that broke.
Out of scope:

No changes to NPCs, Rumors, Quests, Events, Relations, Map, or any tab other than Factions.
Don't add size/alignment filters or sorting anywhere yet.

When done, summarize: (a) the migration logic you wrote, (b) how the alignment/activity/size controls look, (c) how clocks stack in the UI, (d) confirm Relations still works, (e) what I should test.