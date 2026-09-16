# scry build: standing instructions for Copilot

You are building **scry**: a read-optimized code viewer. A single Go binary indexes a workspace and serves a vanilla-JS web UI, with optional language-server navigation and edits delegated to external coding-agent CLIs. The full specification is `SPEC.md`. The work is split into numbered steps; each step's prompt lives in `.github/prompts/NN-*.prompt.md` and defines the scope of the current chat.

## Context discipline (critical: the project is larger than one context window)
- **Never read `SPEC.md` whole.** Run `grep -n '^## \|^### ' SPEC.md` to find line numbers, then read only the sections the step names.
- **Read `HANDOFF.md` first** (if it exists) and trust it. Only open existing source files you must modify, or whose signatures are missing from `HANDOFF.md`. Search with grep and read line ranges rather than whole large files.
- **Never open `web/app.js`.** It is a generated bundle. Edit `web/src/*.js`, then run `node scripts/build-web.js 2>&1 | tail -5`.
- **Keep command output short:** `go test ./... 2>&1 | tail -40`, `go vet ./... 2>&1 | tail -20`. Rerun a failing test with `-run TestName`. Never print whole files or long logs with cat.
- **Stay in scope.** Do only the current step. Don't build later steps' features; leave small, nil-safe extension points instead.
- **Low on context?** If the conversation is getting long before the step is done, stop at the last passing state, update `HANDOFF.md`, commit, and tell the user exactly where to resume.

## Engineering rules
- Go: current stable version, module `scry`, everything in `package main` at the repo root. Allowed dependencies: `github.com/alecthomas/chroma/v2` and `github.com/yuin/goldmark`. Everything else must come from the standard library. No CGO.
- Frontend: vanilla ES modules in `web/src/`, no frameworks, no npm packages. Bundle into `web/app.js` and **commit the bundle**.
- Never write into the served workspace except where the spec's harness and undo rules allow it. Per-user state lives in `$XDG_CONFIG_HOME/scry` or `~/.scry`.
- Every client-supplied path goes through `safePath`/`resolvePath`. Every endpoint that runs a process or writes files uses `localPost`. Never run commands through a shell; pass argv slices.
- Fail quietly when git, language servers, or harnesses are missing: degrade the feature, never panic or return 500.
- Keep JSON field names exactly as in `SPEC.md`. Arrays must serialize as `[]`, never `null`.
- Match existing code style. Comment the *why* of non-obvious decisions, not the what.
- If the spec is ambiguous, pick the simplest behavior consistent with it and record the choice in `HANDOFF.md`.

## Definition of done for every step
1. `go build ./... && go vet ./...` are clean.
2. `go test ./...` passes. New logic in the step has table-driven tests.
3. `node scripts/build-web.js` succeeds whenever frontend files changed.
4. The step's "Verify" commands have been run and their results reported briefly.
5. `HANDOFF.md` has been rewritten in the format below, under 150 lines.
6. `git add -A && git commit -m "Step NN: <title>"`.

## HANDOFF.md format
```
# HANDOFF
## Completed steps
## File map            (one line per file: what it owns)
## Go API              (cross-file functions/types with signatures)
## Frontend API        (exports each web/src module provides; S state fields)
## HTTP endpoints      (method, path, params -> response keys)
## Decisions/deviations from SPEC.md
## Stubs & TODOs for later steps
## Verify              (exact commands)
```
