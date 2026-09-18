---
mode: agent
description: "scry step 18: Codex + GitHub Copilot, per-harness models, concurrent dispatch, -verbose"
---
# Step 18: Multi-harness support

Read `HANDOFF.md` first. Backend and frontend both. This step extends the agent
editing feature built in steps 13–15; it does not replace it.

## Read from SPEC.md
§13 in full (13.1–13.6). Re-skim §4.11 (only the parts §13 says it replaces:
the preset list and the single-flight `errAgentBusy` rule) and the `agentHarness`
JSON shape.

## Build
1. `agent.go`:
   - Add `ModelFlag`, `DefaultModel`, `Models []string` to `agentPreset`; fill them
     in for the existing three presets and add `codex` and `github-copilot`,
     exactly per the table in §13.1.
   - `resolveAgentSpec` gains a `model string` parameter and the insertion logic
     from §13.2 (before the flag preceding `{prompt}` when there is one, otherwise
     before `{prompt}` itself; `{model}` substitution for custom templates).
   - `agentHarness` gains `Models []string` and `Model string`.
   - `discoveredModels`, `discoveringModels`, `discoverHarnessModels`,
     `runModelDiscovery` per §13.3. Write one small parser per harness for
     whatever its "list models" command prints; anything that fails to parse
     just keeps the static list — never surface a discovery error to the caller.
   - Replace the single-flight check in `Start` with `overlapLocked` and the
     double-checked-lock sequence from §13.4: check under `m.mu`, do the
     (potentially blocking) snippet read and git probe outside it, re-acquire
     `m.mu`, check again, only then register the job.
   - `settings{Agent, Models map[string]string}`; `Select(name string, model ...string)`
     persists the chosen model per harness.
2. `main.go`: `-verbose` flag setting `uiVerbose bool` (package-level var, likely
   in `ui.go`). Wire the banner/flag parsing the same way `-quiet` already is.
3. `server.go`: behind `if uiVerbose`, log each request line, non-empty search
   queries, and symbol lookups as they're served. Keep these one-line, so a
   verbose session's output stays scannable.
4. `agent.go`: `uiVerbosePrompt(jobID int64, harness, prompt string, w io.Writer)`,
   called from `run` right before `exec`, gated on `uiVerbose`.
5. `POST /api/agent/select` reads an optional `model=` param and passes it through
   to `Select`.
6. Frontend (`agent.js`):
   - The harness picker (`#agent-pick`) shows a model dropdown next to each
     installed harness, seeded from `harness.models` with `harness.model`
     preselected; changing it calls `apiPost('/api/agent/select', {name, model})`.
   - Footer harness menu (`#agent-menu`) shows the current model next to the
     harness name and lets it be changed the same way.
   - No change to the composer, job polling, or undo flow — those are unaffected
     by concurrency or model choice.

## Tests (`agent_test.go`)
- `resolveAgentSpec` model insertion: for a preset whose `{prompt}` is preceded
  by a flag (e.g. `claude ... -p {prompt}`), the model flag lands before that
  flag; for one where it isn't (e.g. `codex exec ... {prompt}`), it lands
  directly before `{prompt}`. A custom template with `{model}` substitutes it.
  No model requested falls back to `DefaultModel`.
- `Select` with a model persists `{Agent, Models}` to `settings.json`; a new
  manager restores both.
- `discoverHarnessModels`: a fake harness binary that prints a fixed model list
  is picked up on the second call (first call returns the static list); a
  harness whose discovery command fails or is missing keeps the static list;
  concurrent calls for the same harness name trigger only one discovery run
  (assert with a counter in the fake script).
- Concurrent dispatch: two `Start` calls on the same file with disjoint ranges
  both succeed and run at once (assert both `Running` before either finishes,
  using a fake harness that sleeps); overlapping ranges on the same file give
  `errAgentBusy`; the same range on two different files both succeed.
- Race the check: with a fake harness and a slow `readLineRange` substitute (or
  a large file plus a short artificial delay), confirm two dispatches racing
  into the pre-read window don't both get admitted when their ranges overlap —
  run this test with `-race`.
- `-verbose`: with `uiVerbose` set, `run` calls `uiVerbosePrompt`; capture it
  with a writer substitute and assert the exact prompt text appears.

## Verify
```
go test ./... -race -run 'Agent|Verbose' 2>&1 | tail -40
node scripts/build-web.js 2>&1 | tail -5
go test ./... 2>&1 | tail -10
```
Manually: start with two harnesses installed (or two fake scripts registered as
custom templates), select different models for each from the picker, and fire
edits on disjoint ranges of the same open file — both should run and finish
independently, and the footer should reflect the model in use.

Then follow the definition of done in copilot-instructions.md. Also touch up
`README.md`'s harness table and `ACCEPTANCE.md` (from step 17) to cover the
§13.6 checklist items, since step 17 already ran before this step existed.

Final commit: "Step 18: multi-harness support".
