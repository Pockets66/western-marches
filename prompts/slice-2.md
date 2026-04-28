Slice 2: localize fonts for offline use.
I've added a fonts/ directory at the project root containing .woff2 files for Cinzel (weights 400, 600, 900) and Crimson Pro (weights 300, 400, 600, plus 300 italic and 400 italic). Also an OFL license file.
Please do the following:

Remove the Google Fonts <link> from index.html and replace it with @font-face rules at the top of the <style> block, one rule per weight/style. Reference the actual filenames in fonts/. Use font-display: swap. Keep the existing font-family names ("Cinzel" and "Crimson Pro") unchanged so the rest of the CSS still works.
List the filenames in fonts/ yourself rather than guessing — the version numbers vary.
Update CLAUDE.md: under "Architecture constraints," strengthen the rule to say "No external runtime dependencies at all. Fonts are bundled locally. The app must work fully offline from file://."
Do not change any other code. No features, no refactoring.

When done, summarize: (a) how many @font-face rules you wrote, (b) the updated CLAUDE.md constraint, (c) what I should test.