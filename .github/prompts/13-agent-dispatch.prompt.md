---
mode: agent
description: "scry step 13: coding-harness discovery, selection, dispatch, change detection"
---
# Step 13: Agent dispatch (backend)

Read `HANDOFF.md` first. Backend only. Undo comes in step 14, but leave the hook for it: call `capturePreEdit` and `planUndo` from functions defined in `agent_undo.go` as **minimal stubs** (return an empty `preEdit` holding only `status`, and a nil plan) so step 14 fills them in.

## Read from SPEC.md
§4.11, §4.13 (rows `/api/agent/*` and the agent keys of `/api/meta`).

## Build
1. `settings.go`: `settings{Agent}`, `settingsPath`, `readSettings` (never fails), `writeSettings` (mutex, mkdir, indented JSON plus newline).
2. `agent.go`:
   - `agentTimeout` (a package var so tests can shorten it) and `agentLogBytes`.
   - `agentPreset`, `agentPresets`, `agentHarness`, `agentJob` (JSON tags per §4.11, with unexported `out`, `stderr`, `start`).
   - Errors `errAgentBusy`, `errAgentDirty` (its message must contain "uncommitted"), `errAgentNone`.
   - `agentManager`; `newAgentManager(root, flagSpec, lsp)`; `resolveAgentSpec`; `agentPresetNames`; `Detect`; nil-safe `Name` and `Pinned`; `Select`.
   - `Job` (snapshot copy with log, stdout, stderr, live ms); `Start`; `run`; `settle`; `Cancel`; `Close`.
   - `changedSince`, `worktreeSnapshot`, `changedSinceMaps`, `lineRef`, `readLineRange`, `agentPrompt` (exact format from §4.11).
   - Reuse the `tailBuffer` from `lspsetup.go`.
   - Handlers: `agentOrFail`, `handleAgentHarnesses`, `handleAgentSelect`, `handleAgentEdit`, `handleAgentJob`, `handleAgentCancel`. Every mutation goes through `localPost`. Status codes: 409 for busy or dirty, 400 otherwise, 404 when disabled.
3. `main.go`: `-agent` (fatal on a bad spec), `-no-agent`, `pxSrv.SetAgent`, the banner line `agent  <name> (edits this workspace)`, `agent.Close()` on exit. `/api/meta`: `agent`, `agentPinned`, `agents`.
4. Log each dispatch, finish, and failure to the terminal with `uiStatus`, as §4.11 describes.

## Tests (`agent_test.go`)
Use a **fake harness**: a shell script written into `t.TempDir()`, used via the template `"<script> {prompt}"`. It reads env vars to decide what to do: append a line to file X, create file Y, sleep, print to stderr and exit 1. It also writes the received prompt to a file for assertions. Skip on Windows.
- `resolveAgentSpec`: preset names (case-insensitive) resolve only when their binary exists (put a fake `claude` on PATH in a temp dir); a template without `{prompt}` is an error listing the presets; a missing binary is an error.
- Select persists the spec to settings under a temp `XDG_CONFIG_HOME`; a new manager restores it; a pinned manager refuses Select.
- Prompt format matches exactly, including the extension fence and `l1-l2` versus a single line.
- Start refuses an empty instruction and a missing harness; a second Start while running gives `errAgentBusy`.
- In a temp git repo, a dirty target file gives `errAgentDirty` with "uncommitted"; `force` proceeds.
- Change detection: the harness edits a clean file → `Changed` has it; the harness edits a file that was **already** modified → still detected; the harness creates a new file → detected; `Tracked` is true in git and false outside it.
- Failure: exit 1 gives `Error` set and stderr captured in the job snapshot.
- Timeout: with `agentTimeout` set to 200 ms and a sleeping harness, the error mentions "gave up".
- Cancel stops a sleeping harness.
- HTTP: every agent endpoint returns 404 when the agent is nil; POST edit without a matching Origin returns 403; GET job returns `{idle:true}` before any run.

## Verify
```
go test ./... -run Agent 2>&1 | tail -30
```
Then follow the definition of done in copilot-instructions.md.
