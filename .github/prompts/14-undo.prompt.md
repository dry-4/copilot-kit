---
mode: agent
description: "scry step 14: undo of the last agent edit"
---
# Step 14: Undo

Read `HANDOFF.md` first. Backend only. Replace the step 13 stubs in `agent_undo.go`.

## Read from SPEC.md
§4.12, plus the `/api/agent/undo` row of §4.13.

## Build
`agent_undo.go`:
- `undoFileBytes` (8 MB) and `undoTotalBytes` (64 MB) as package vars so tests can lower them.
- `errUndoNone`, `errUndoStale`.
- Types `preEdit{status, head, saved, missing}`, `savedFile`, `undoStep`, `agentUndo{steps, stamps}`.
- `capturePreEdit(root)` per §4.12.
- `planUndo(root, pre, changed) (*agentUndo, string)`, all or nothing, per the case table in §4.12.
- `headBlob(root, head, rel)` using `git -C root ls-tree -z` and `git cat-file blob <head>:./<rel>`, which resolve relative to the served root even when it is a subdirectory.
- `pathStamp`.
- `(m *agentManager) Undo(force)`: busy check, stale check unless forced, consume the plan **before** applying, apply, `settle`, logging.
- `restoreFile`, `handleAgentUndo` (`localPost`; 404/409/500 mapping).

Make sure `run` in `agent.go` sets `job.Undoable` and `job.UndoNote` from `planUndo`, and that `Start` clears `m.undo`.

## Tests (temp git repos, fake harness from step 13)
- A clean tracked file edited by the harness is restored byte-for-byte from HEAD, with mode 0755 preserved for an executable file.
- A dirty file (uncommitted edit before the run, `force` start) is restored to the pre-run dirty content, not HEAD.
- A new file created by the harness is removed. A new untracked directory with files is removed entirely.
- A file deleted by the harness is restored.
- An untracked directory that existed before and gained a file gives `Undoable:false` with a note.
- A dirty file larger than `undoFileBytes` (lowered for the test) gives `Undoable:false` with a note.
- Stale: modify a changed file after the run, and `Undo(false)` gives `errUndoStale` naming the path. `Undo(true)` restores anyway.
- Single use: a second `Undo` gives `errUndoNone`. A new `Start` discards the plan.
- Served root as a subdirectory of the repo: a HEAD blob restore works.
- HTTP: POST undo with a proper Origin returns `{undone:[...]}`; no plan returns 404; stale returns 409.

## Verify
```
go test ./... -run 'Agent|Undo' 2>&1 | tail -30
```
Then follow the definition of done in copilot-instructions.md.
