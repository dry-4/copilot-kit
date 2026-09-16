---
mode: agent
description: "scry step 01: project skeleton, CLI, HTTP server core"
---
# Step 01: Skeleton, CLI, server core

No code exists yet, only `SPEC.md` and `.github/`.

## Read from SPEC.md
§1, §2, §3, §4.1, §4.2, §4.13 (only: ServeHTTP wrapper, scavenger, writeJSON/fail, safePath/resolvePath, and the rows `/`, `/static/*`, `/static/themes.css`, `/api/meta`), §5.1.

## Build
1. `git init` if needed. Create `go.mod` (module `scry`, current stable Go), `VERSION` (0.1.0), MIT `LICENSE`, and `.gitignore` (the `scry` binary, `dist/`, `bench-repos/`).
2. `main.go`:
   - Parse all flags from §4.1. Flags for later features are stored in package variables and do nothing yet.
   - `version` subcommand and `-version`/`-v`.
   - `resolveTarget` and `splitTargetLine`. Git top-level detection shells out to `git rev-parse --show-toplevel` through a small helper `gitProbe` in `git.go` that also honors `-no-git`.
   - `viewerURL`, `listen` (port walking), `openBrowser` (including WSL), signal handling (graceful, then a second signal exits 130).
3. `ui.go` plus `tty_linux.go`, `tty_bsd.go`, `tty_other.go`, each with build tags.
4. `server.go`:
   - `//go:embed web`, and `-dev` disk assets through `useDiskAssets`.
   - `Server{ix, lsp, agent, mux, lastReq}` with the gzip pool, scavenger, `safePath`, `resolvePath`, `writeJSON` (no HTML escaping), and `fail`.
   - Handlers: `/`, `/static/`, `/static/themes.css`, `/api/meta` (return all keys from §4.13 with zero values for features not built yet).
   - A stub `lspManager` in `lspservers.go` with `Allowed(abs) bool { return false }`, `Available() []string`, and `Close()`. A nil-safe `*agentManager` placeholder type in `agent.go` with `Name()` and `Pinned()`.
   - `Index` does not exist yet. Create a minimal `index.go` with `NewIndex(root)`, `Root()`, `Ready()`, `Stats()` so it compiles; step 02 replaces it.
5. `web/index.html`: every region and ID from §5.1 as placeholders. `web/style.css` with base layout variables. `web/themes/dark.css` with one theme. `web/src/main.js` with a boot that fetches `/api/meta` and shows the workspace name.
6. `scripts/build-web.js` exactly per §6 (Bun if present, otherwise the Node fallback with duplicate top-level name detection). Run it and commit `web/app.js`.

## Tests
- `resolveTarget`: a directory; a file inside a temp git repo (skip if git is missing); `file:42`; `file:42:7`; a file whose real name contains a colon.
- `listen`: occupy a port, then confirm the next one is chosen.
- `safePath`: rejects `../x`, `a/../../x`, `/etc/passwd`; accepts `a/b`, `.`, `/a/b` (leading slash trimmed).
- HTTP tests with `httptest`: `/` returns HTML, `/api/meta` returns JSON with the expected keys, a gzip response when requested, `Cache-Control: no-store`.

## Verify
```
go run . -no-open -port 0 .   # prints URL; then in another terminal:
curl -s localhost:<port>/api/meta | head -c 400
```
Then follow the definition of done in copilot-instructions.md.
