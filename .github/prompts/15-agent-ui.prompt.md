---
mode: agent
description: "scry step 15: selection bar, right-click menu, agent composer, undo button"
---
# Step 15: Selection actions and agent UI

Read `HANDOFF.md` first. Frontend step.

## Read from SPEC.md
§5.2 (only `selbar.js` and `agent.js`), §5.1 (`#agentbox`, `#sel-menu`, `#agent-menu`, `#footer-sel`, `#footer-actions`), §4.11 (only the prompt format, for Copy for Agent).

## Build
1. `selbar.js`:
   - `updateSelectionBar()` on `selectionchange`: when the selection is inside the code view or diff view, the left part of the status bar is replaced by `#footer-sel` buttons: **Copy Ref** (`path:l1-l2`), **Copy for Agent** (the `### Reference` block with the fenced snippet, the same shape as `agentPrompt` minus the instruction section), **Edit with Agent**, **Find Usages**. Hide the bar when the selection is cleared.
   - Line range from code view selections via `toPos`.
   - `diffSelection()` for the diff view: use `data-new`, `data-old`, `data-type`, `data-side` from step 07 to map to working-tree lines. Deleted lines map to the nearest following new line (or the previous one at the end of a hunk). The range covers the new-side lines touched.
   - Snippet text for Copy for Agent comes from the doc's loaded lines (strip HTML via textContent) or `/api/raw` for unloaded ranges.
   - Right-click on a selection opens `#sel-menu` at the pointer, rebuilt from the `#footer-sel` buttons each time; Esc or an outside click closes it.
   - `Alt+C`, `Alt+A`, `Alt+U`, `Alt+E` assign the step 06 hooks. `selectAll`, `clearSelectAll`, `copySelectAll` for whole-file selection in the virtual view (`Mod+A` inside the editor selects the logical whole file; copy fetches `/api/raw`).
2. `agent.js`:
   - `applyAgentMeta()`: from `S.meta`, the footer `Agent: <name>` button; `Agent: choose` when nothing is selected; hidden when the agent is unavailable (the endpoint returns 404 or `meta.agents` is empty and not pinned).
   - `openAgentEdit({path, l1, l2, fromDiff})`: show `#agentbox` anchored near the selection with `#agent-ref` (`path:l1-l2`). If nothing is selected, show the `#agent-pick` picker first (`/api/agent/harnesses`): installed harnesses can be selected, missing ones are listed with their command text and the settings path. Choosing one runs `apiPost /api/agent/select`.
   - `submit()` on Enter (Shift+Enter inserts a newline): `apiPost /api/agent/edit`. On an error whose message contains "uncommitted", `confirm()` and then retry with `force:1`. Edits started from the diff view always send `force:1`.
   - `tick()` polls `/api/agent/job` every 700 ms while running, showing "Running <harness>… Ns" and a Cancel button (`/api/agent/cancel`).
   - `finish(job)`: on `error`, show `#agent-err` with the message plus collapsible **stdout** and **stderr** blocks, and keep the composer and instruction. On success, close the composer and call `reloadWorkspace()`.
   - `reloadWorkspace()`: `apiPost /api/reindex`, then `reloadOpenTabs()` (each tab stays in its mode: source, diff, or preview; refresh the gutter and an open diff), redraw the tree keeping expansion, and toast "Changed N files: a, b" (or "Reloaded" when `tracked` is false).
   - `setUndo(job)`: when `undoable`, show **Undo Edit** in `#footer-actions`; otherwise, when files changed, a disabled button with `undoNote` as its tooltip.
   - `undo()`: `apiPost /api/agent/undo`. On 409 "changed since", `confirm()` and then `force:1`. Then `reloadWorkspace()` and hide the button.
   - Footer harness menu `#agent-menu`: list harnesses, switch on click, disabled when pinned.
   - `initAgent()`: wire everything; restore a running job's state on page load (poll once).
3. Styles for `#agentbox`, the menus, the error blocks, and the footer buttons.

## Tests
Keep the Go tests green and rebuild the bundle.

## Verify (manual; report)
Create `fake.sh`:
```sh
#!/bin/sh
printf '%s\n' "$1" > /tmp/scry-last-prompt.txt
echo "// edited by fake agent" >> "$(printf '%s' "$1" | sed -n 's/^### Reference: \([^:]*\):.*/\1/p')"
```
Then `chmod +x fake.sh` and run `go run . -agent "./fake.sh {prompt}" .` in a git repo.
1. Select lines in the source view, press `Alt+E`, submit. The file reloads with the appended line; Undo Edit appears; undo restores it.
2. Select in the split diff view, including a deleted line, then Edit. The range is correct in `/tmp/scry-last-prompt.txt`.
3. Try a dirty file from the source view: a confirmation dialog appears.
4. Make the script `exit 1` with stderr output: the inline error shows stderr and the instruction is kept.
5. Copy Ref and Copy for Agent put the correct text on the clipboard; the right-click menu works.
6. Without `-agent`, the picker appears and the choice persists to `~/.scry/settings.json`.

Then follow the definition of done in copilot-instructions.md.
