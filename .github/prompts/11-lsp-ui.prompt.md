---
mode: agent
description: "scry step 11: go-to-definition, references, hover cards, call trails, LSP setup panel"
---
# Step 11: Language server UI

Read `HANDOFF.md` first. Frontend step.

## Read from SPEC.md
§5.2 (only `lsp.js`, `hover.js`, `inspector.js`, `calls.js`, `lspsetup.js`, plus the LSP state part of `status.js`), and the `/api/lsp/*` rows of §4.13 for response shapes.

## Build
1. `lsp.js`:
   - `warmLSP(doc)` calls `/api/lsp/warm` on open and then polls with backoff while the state is `starting` or `indexing`, updating `S.lsp` and the status bar.
   - `gotoDefinition()` (`F12`, and `Mod+Click` via `cursor.js`): when the LSP is ready, use `/api/lsp/def` with the UTF-16 column; if it returns nothing or no server exists, fall back to `/api/def?sym=<word>`. One hit jumps (pushing history); several open a small picker using the palette overlay.
   - `findReferences()` (`Shift+F12`, plus `Alt+U` Find Usages later) sends results to the inspector. Without an LSP it falls back to a whole-word `/api/search`.
   - External paths (`ext:true`) open as read-only tabs labeled with their absolute path.
   - Assign the `hooks.*` from step 06.
2. `hover.js`:
   - A 250 ms debounced hover over an identifier (`wordAtPoint` from `cursor.js`) calls `/api/lsp/hover`, and shows `#hovercard` with the highlighted signature, doc text (escaped, paragraphs), and Definition / References buttons. It hides on mouse-out, scroll, or Esc.
   - While `Mod` is held, underline the identifier under the pointer as a link.
3. `inspector.js`: a right-side panel with a resizer (persisted) and tabs for References and Call Trail. `inspectReferences(hits)` groups hits by file with snippets; clicking one jumps there.
4. `calls.js`:
   - `showCalls()` (`Alt+Shift+H`) prepares from the caret via `/api/lsp/calls`.
   - A tree with an incoming/outgoing toggle; each node expands lazily with `item` and `dir` (a spinner while loading); clicking a node jumps to its first site.
   - A message when there is no server.
5. `lspsetup.js`: when `lsp.state` is `off` and `missing` is set, the status bar shows **"LSP: set up"**. Clicking it opens a panel with `/api/lsp/setup` data:
   - auto options have an Install button (`apiPost /api/lsp/install`) and poll the job output
   - manual options show the command with a copy button
   - "Detect & start" calls `apiPost /api/lsp/start`
   - errors are shown inline

   A 403 from `localPost` shows its message.
6. `outline.js`: when the LSP becomes ready, fetch `/api/lsp/symbols` and call `upgradeOutline`.
7. Add the new commands to the palette.

## Tests
Keep the Go tests green and rebuild the bundle.

## Verify (manual; report; needs at least one server, e.g. `go install golang.org/x/tools/gopls@latest`)
In a Go repo:
- F12 on a local function and on a stdlib function (it opens the external read-only file)
- Shift+F12 references
- a hover card
- `Alt+Shift+H` call trail, expanding two levels
- the status LSP state moves from starting to ready
- on a `.rs` file without rust-analyzer, "LSP: set up" appears and the panel lists the recipes

Then follow the definition of done in copilot-instructions.md.
