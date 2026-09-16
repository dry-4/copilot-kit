---
mode: agent
description: "scry step 07: git status, badges, gutter, diff view"
---
# Step 07: Git integration and diff view

Read `HANDOFF.md` first.

## Read from SPEC.md
§4.9, §4.13 (rows `/api/diff`, `/api/gutter`, and the `diffAvailable` field of `/api/file`), §5.2 (`diff.js`, plus the tree badge and changed-filter parts of `tree.js`).

## Build
1. `git.go`: replace the stubs.
   - `gitDisabled` and a memoized `gitProbe`/`gitAvailable`.
   - `gitStatus`: porcelain v2 `-z`, parsing records `1`, `2` (rename consumes the next NUL field), `u`, `?`; `mapXY`; paths converted relative to the served root, which may be a subdirectory of the repo top-level (compute the prefix from `rev-parse --show-prefix`, or compare against the top-level); nil when empty.
   - `gitDiff`, `gitHunks`, `parseNewStart`.
2. The index overlay now works. Confirm `Status` and `Dirty` appear in `/api/tree`.
3. `server.go`: `/api/diff`, `/api/gutter` (arrays never null), `diffAvailable` in `/api/file`, `git` in `/api/meta`.
4. Frontend:
   - `tree.js`: colored status letters, dirty dots on folders, and a `#btn-changed` filter (shown only when `meta.git`) that shows only changed files and dirty folders.
   - Gutter: `tabs.js` loads `/api/gutter` for each opened doc; a renderer decorator draws added, modified, and deleted markers.
   - `diff.js`:
     - `#diffview` over the editor for the active tab, toggled by `Mod+D` (assign `hooks.toggleDiff`)
     - `parseDiff` into hunks
     - split and unified tables with old and new line numbers and +/- markers, highlighted lines (reuse the classes by requesting `/api/file` chunks for new-side lines where practical; plain escaped text is acceptable for deleted lines)
     - a layout switch remembered in localStorage
     - a toast when there's no diff

     **Each diff row element must carry `data-old`, `data-new`, and `data-type` (`add`/`del`/`ctx`), and split cells carry `data-side`.** Step 15 maps selections using them.
   - Add diff commands to the palette.

## Tests (temp repos with `git init`, user.name/email set; skip if git missing)
- Status: modified, added (staged), deleted, untracked, renamed (staged rename), conflict if practical; served root as a subdirectory gives correctly relative paths and excludes files outside it.
- `-no-git` (set `gitDisabled`) gives nil and `gitAvailable` false.
- `gitHunks`: a pure insertion gives `added`; a line replacement gives `modified`; a pure deletion gives a `deleted` marker at the preceding line (0 for the top of the file).
- HTTP: `/api/diff` `available` is false for a clean file and true after a change.
- Index: a modified file's node has `status:"M"` and its parent directory has `dirty:true`.

## Verify (manual; report)
Modify, add, and delete files in a git repo, click reindex: badges, dirty folders, and the changed filter work; the gutter markers are right; `Mod+D` shows split and unified diffs.

Then follow the definition of done in copilot-instructions.md.
