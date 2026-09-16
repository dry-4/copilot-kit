---
mode: agent
description: "scry step 02: gitignore engine, workspace index, file tree UI"
---
# Step 02: Index, gitignore, file tree

Read `HANDOFF.md` first.

## Read from SPEC.md
§4.3, §4.13 (rows `/api/tree`, `/api/reindex`), §5.2 (only `state.js`, `ui.js`, `tree.js`, `panels.js`, `main.js`).

## Build
1. `ignore.go`: a full gitignore engine.
   - Comments, trailing-space trimming, `!` negation (last match wins), `/` anchoring, patterns containing a slash are anchored, trailing `/` means directory-only, `*`, `?`, `[...]`, `**`.
   - A literal fast path, with a regex otherwise.
   - `newIgnoreSet`, `child(patterns)`, `match(rel, isDir)`, `readGitignore(dir, relDir)` (patterns scoped to that directory), and `compilePattern(p) (rule, bool)` plus `rule.hit(rel, isDir)`. Step 05 reuses the last two for search globs.
2. `index.go`: replace the stub with the full implementation per §4.3.
   - `FileEntry`, `Node`, `Index`, `Build`, `Children`, `underIgnoredLocked`, `listIgnored`, `sortNodes`, `WaitReady`, `Files`.
   - A semaphore of `NumCPU*4` with an inline fallback when it's full. Publish the root listing early.
   - Git overlay: call `gitStatus(root)` in parallel with the walk. Put a stub in `git.go` returning `nil` (step 07 implements it), but write the Status/Dirty overlay code now.
3. `server.go`: `/api/tree` (the 300 ms wait loop while not ready, 404 for an unknown directory) and `/api/reindex` (**POST only**). In `main.go`, start `go ix.Build()` and print the file count and duration.
4. Frontend:
   - `state.js`: `S`, `$`, `$$`, `esc`, `api`, `apiPost`, `debounce`, `isMac`, `MOD`, constants.
   - `ui.js`: shared DOM references, `showToast`, `copyToClipboard`.
   - `tree.js`: lazy expansion, ignored entries dimmed, rendering for status badges and dirty dots (data is empty for now), a click handler that calls `openFile` through an import hook that step 04 fills in. For now, log or toast the path.
   - `panels.js`: rail/sidebar panel switching, a draggable resizer persisted in localStorage, a reindex button calling `apiPost('/api/reindex')` then redrawing the tree.
   - `main.js` boot: `/api/meta`, root name, draw the tree, poll meta every 150 ms until `ready`.
   - Enough `style.css` for the rail, sidebar, and tree.

## Tests
- ignore: a table of (patterns, path, isDir) to expected result covering negation re-including a file, `/root-only`, `dir/`, `**/foo`, `a/**/b`, `*.log` at any depth, character classes, and a nested `.gitignore` overriding its parent.
- index (temp directory):
  - ignored files are listed with `ignored:true` but absent from `Files()`
  - `.git` is not listed at all
  - symlinks are skipped
  - `Children` of a path inside an ignored directory lists entries from disk, all ignored
  - `Children("a/../..")` is refused
  - `Ready()` flips after `Build`
- HTTP: `/api/tree` root and a subdirectory; GET `/api/reindex` returns 405.

## Verify
```
go run . -no-open -port 7777 .    # then open http://127.0.0.1:7777 in a browser: tree expands, ignored entries dimmed
```
Then follow the definition of done in copilot-instructions.md.
