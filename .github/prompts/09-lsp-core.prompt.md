---
mode: agent
description: "scry step 09: LSP JSON-RPC client, server registry, discovery, lifecycle"
---
# Step 09: Language server core

Read `HANDOFF.md` first. Backend only.

## Read from SPEC.md
§4.10: only the Registry table, `lspBinDirs`, Client, and Manager paragraphs. Skip Navigation, Call trails, and Setup; those are step 10.

## Build
1. `lsp.go`:
   - Types: `lspPosition`, `lspRange`, `lspLocation`, `lspLocationLink`, `lspDocumentSymbol`, `rpcMessage`, `rpcError`.
   - `lspClient{def, root, cmd, stdin, reader, mu, nextID, pending map[id]chan, opened map[abs]version, progress set, dead error}`.
   - `start(ctx)`: spawn with argv (no shell), discard stderr, `readLoop`, `initialize` (rootUri, workspaceFolders, capabilities per §4.10, `InitOptions`), then `initialized`.
   - `readFrame` / `write` with `Content-Length` framing. `call(ctx, method, params, out)` correlates by id and handles ctx cancellation (sends `$/cancelRequest`). `notify`.
   - `reply` answers server→client requests: `workspace/configuration` gets an array of nulls sized to the items; `window/workDoneProgress/create`, `client/registerCapability`, and anything else get `null`.
   - `onNotification` tracks `$/progress` begin/end tokens, which drive `busy()`.
   - `ensureOpen(abs, rel)` sends `didOpen` once with the languageId. `closeDoc(abs)` sends `didClose`.
   - `toLSP(lineText, line, byteCol)` and `fromLSP(lines, pos)` do UTF-16 ↔ byte conversion. `pathToURI` / `uriToPath` handle Windows drive letters and percent-encoding.
   - `alive()`, `fail(err)` (fails all pending calls), `shutdown()` (shutdown, exit, wait up to 2 s, then kill).
2. `lspservers.go`: replace the stub.
   - `lspServerDef`, `LanguageID`, the full `lspRegistry` table with install recipes (the `lspInstall` type is `{OS string, Cmd []string, Auto bool}`, used in step 10).
   - `lspState` constants, `maxLSPRestarts`.
   - `lspManager`: `allow` / `Allowed` (cap 20k), `newLSPManager` (background `discover` when enabled), `discover` (atomic swap), `isDiscovered`, `lspBinDirs`, `lookPathIn`, `Available`, `defFor`, `State`, `client` (dedup `starting` channel, restart up to 3, failed map), `spawn` (30 s), `CloseDoc`, `Close`, `Rescan` (rediscover and clear `failed`), and `errNoServer`.
3. `main.go`: honor `-no-lsp`; call `lsp.Close()` on exit; print the available servers after indexing. `/api/meta` `lspServers`.
4. `server.go`: add an `lspBrief(rel)` helper returning `{state, server}` (step 10 adds a `missing` language) and use it in `/api/file`; `/api/def`'s `lsp` field uses `State`.

## Tests
- Framing: write then read round trip; multiple frames in one buffer; a malformed header gives an error.
- UTF-16: `toLSP`/`fromLSP` on `"a😀b"` (the emoji is 4 bytes, 2 UTF-16 units), CJK text, and a column past the end of the line clamps.
- `pathToURI`/`uriToPath` round trip, including spaces.
- Registry precedence: with fake binaries for both `pyright-langserver` and `pylsp` placed in a temp dir on PATH, `.py` resolves to pyright; with only `pylsp`, to pylsp.
- **Fake language server:** in `lsp_test.go`, `TestMain` checks an env var (for example `SCRY_FAKE_LSP=1`). When it is set, the test binary acts as a minimal stdio LSP server:
  - answers `initialize`, `textDocument/definition` (a fixed location), and `shutdown`
  - emits a progress begin/end pair
  - on `SCRY_FAKE_LSP_CRASH=1`, exits after initialize

  Point a registry def at `os.Args[0]` with that env. Test that:
  - initialize and definition round trip
  - `busy()` toggles with progress
  - a crash leads to a respawn on the next `client()` call
  - after 3 crashes, the error is returned and no further spawn happens
  - concurrent `client()` calls spawn only once

## Verify
```
go test ./... -run 'LSP|Frame|UTF16|URI|Registry' 2>&1 | tail -20
```
If `gopls` is installed, add a quick manual check: run scry on a Go repo, open a `.go` file; `/api/file` reports `lsp.state` going from `starting` to `ready`.

Then follow the definition of done in copilot-instructions.md.
