---
mode: agent
description: "scry step 06: palette, search panel, outline, in-file find, keyboard shortcuts"
---
# Step 06: Navigation UI and shortcuts

Read `HANDOFF.md` first. Frontend step.

## Read from SPEC.md
§5.2 (only `palette.js`, `search.js`, `outline.js`, `find.js`, `shortcuts.js`, and the key-label helpers in `state.js`), §5.3 (the full table).

## Build
1. `state.js` key labels: `keyParts`, `keyLabel("Mod+Shift+F")` (Mac shows `⌘⇧F`, others `Ctrl+Shift+F`), support for the `"pc|mac"` combo syntax, `keyCaps`, `withKeys`, and `applyKeyLabels()`, which fills every `[data-keys]` element and tooltip.
2. `palette.js`: one overlay (`#overlay` > `#palette`) with modes selected by prefix:
   - none: files via `/api/find`, with matched characters highlighted from `pos`
   - `>`: commands. `COMMANDS` is one array of `{id, title, keys, run}` covering every action that exists, and is extended by later steps.
   - `@`: symbols in the active file via `/api/outline`, fuzzy-filtered on the client
   - `:`: go to line

   Arrow keys, Enter, Esc. Opened by `Mod+K`, `Mod+P` (files), `Mod+Shift+P` (`>`), `Mod+Shift+O` (`@`), `Mod+G` (`:`).
3. `search.js`: the `#panel-search` input, regex/case/word toggle buttons, a glob input, debounced search via `/api/search`, results grouped by file with pre/mid/post rendering (escaped) and a declaration badge; clicking opens the file at that line with the match highlighted. `Mod+Shift+F` focuses it and seeds it with the selection.
4. `outline.js`: `#panel-outline` loads `/api/outline` for the active tab, shows kind badges and indentation, highlights the current symbol while scrolling, and jumps on click. Export `loadOutline` and an `upgradeOutline(symbols)` hook for the LSP step.
5. `find.js`: `#findbar` (`Mod+F`) seeded with the selection; case and regex toggles; searches the active doc through `/api/search` with `glob` set to the exact path; next and previous with Enter/Shift+Enter; the hit count; hit decorations via the renderer decorator hook; minimap dots on the scrollbar track; Esc closes.
6. `shortcuts.js`: one keydown handler driven by a single table for every §5.3 shortcut. Actions that belong to later steps (definition, references, call trail, diff, markdown toggle, selection actions, agent edit) call exported no-op hooks from a small `hooks.js` module (`hooks.gotoDefinition = () => {}` and so on) that later steps assign. The `?` help sheet (`#helpsheet`) is generated from the same table with OS-correct labels. `Mod+B` toggles the sidebar; tab shortcuts `Alt+W`, `Ctrl+Tab`, `Alt+1..9`. Don't capture keys while typing in inputs, except Esc and palette keys.

## Tests
Keep the Go tests green. Rebuild the bundle; the bundler's duplicate-name check must pass.

## Verify (manual; report)
In the browser, exercise every shortcut from §5.3 that is implemented so far: palette in all four modes, workspace search with each toggle, outline click, find next and previous, and a help sheet with Mac or PC labels matching your OS. Confirm that shortcuts for later features are harmless no-ops.

Then follow the definition of done in copilot-instructions.md.
