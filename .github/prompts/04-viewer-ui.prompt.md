---
mode: agent
description: "scry step 04: virtualized code viewer, tabs, status bar, themes"
---
# Step 04: Virtualized viewer UI and themes

Read `HANDOFF.md` first. Frontend-focused step. This is the riskiest UI step, so take care with scrolling performance.

## Read from SPEC.md
§5.1, §5.2 (only `renderer.js`, `tabs.js`, `history.js`, `cursor.js`, `status.js`, `theme.js`, `main.js`), §5.4, §4.13 (row `/api/file` for the response shape).

## Build
1. `renderer.js`:
   - `measure()` gets line height and character width from a hidden probe element.
   - `layout()` sets `#sizer` height to `total*LH` and width from `maxCols` (width is irrelevant when wrap is on).
   - `render()` / `paint()` use rAF-throttled scrolling and mount only rows in view plus `OVERSCAN`. Reuse row elements. Each row has a line-number gutter cell and a code cell holding the server's HTML.
   - `ensureChunks(doc, first, last)` fetches missing 1,000-line chunks once each (dedupe in-flight requests).
   - `refineChunk` re-fetches `refine:true` chunks after 800 ms, retrying a bounded number of times.
   - `decorate(first, last)` is a hook other modules register decorators into (find hits, gutter, occurrences, selection).
   - `toPos`/`toPoint` map between DOM and `{line, col}`. `placeCaret`.
   - `toggleWordWrap` (`Alt+Z`) and `toggleLineNumbers` (`Alt+L`), persisted.
   - Word wrap needs variable row heights. Keep it correct but simple: when wrap is on, measure rendered rows and keep a height cache with prefix sums for visible ranges. Record the approach in HANDOFF.
2. `tabs.js`:
   - `openFile(path, {line, col})` fetches chunk 0, creates or reuses a doc in `S.docs`, pushes history, renders, and centers the line.
   - `closeTab` calls `/api/close`. `reopenClosedTab` (`Alt+Shift+T`). `switchTab`, `drawTabs` (with close buttons), `drawCrumbs`, `showImage`/`hideImage` (via `/api/raw`).
   - `reloadOpenTabs()` re-fetches the visible chunks of every open doc in place, keeping scroll position. Later steps use it after edits.
   - Connect the tree click hook from step 02 to `openFile`, and add `revealFile(path)` to `tree.js`.
3. `history.js`: back/forward stack (`Alt+Left`, `Alt+Right`).
4. `cursor.js`: click places the caret; Left/Right/Home/End; `Mod+Up`/`Mod+Down` (Mac) or `Ctrl+Home`/`Ctrl+End`; double-click selects a word; occurrences of the word under the caret are highlighted in visible rows through a decorator.
5. `status.js`: language, lines, size, cursor position, index time and file count, memory from `/api/metrics` refreshed every 5 s while visible, an LSP state placeholder, and `fitStatus()` adding `fit-1`..`fit-6` classes based on overflow.
6. `theme.js` plus **all 14 theme files** in `web/themes/` (§5.4 list), each defining the same token set. Theme discovery scans loaded stylesheet rules for `[data-theme="…"]`. Persist the choice. Wire the theme button.
7. `main.js`: restore prefs, handle `?path=&line=` then strip it with `history.replaceState`, re-measure after `document.fonts.ready`. Add an `#empty` welcome screen when no tab is open.
8. `style.css`: tabs, crumbs, editor rows, gutter, status bar, token classes mapped to variables, dense monospace layout.

## Tests
There is no JS test framework. Instead:
- Keep the Go tests green.
- Add a Go HTTP test that `/static/app.js` and `/static/themes.css` are served, and that `themes.css` contains all 14 `data-theme` names.

## Verify (manual; report results)
1. `go run . -no-open .`, then open the URL.
2. Open a large file. To make one: `mkdir -p /tmp/big && for i in $(seq 1 200000); do echo "func f$i() { /* $i */ }"; done > /tmp/big/big.go`, then run `go run . /tmp/big/big.go`. Scroll to the end quickly: highlighting is correct, scrolling stays smooth, memory stays reasonable.
3. Toggle wrap and line numbers, switch themes, close and reopen a tab, use back/forward.

Then follow the definition of done in copilot-instructions.md.
