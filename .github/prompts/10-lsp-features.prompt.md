---
mode: agent
description: "scry step 10: LSP navigation, hover, symbols, call hierarchy, install flow, localPost"
---
# Step 10: LSP features and install flow (backend)

Read `HANDOFF.md` first. Backend only.

## Read from SPEC.md
§4.10 (Navigation, Call trails, Setup, `localPost`), §4.13 (all `/api/lsp/*` rows).

## Build
1. `lspnav.go`:
   - `NavHit{path, line, col, endCol, text, ext?, def?}`, `utf16ToByte`.
   - `resolve(client, locs, markDef)` reads target lines from disk for snippets and converts columns. For targets outside the root, call `allow(abs)`, return the absolute slash path, and set `ext:true`.
   - `locate(ctx, method, abs, rel, line, u16col, extra)`: ensureOpen, call, and decode Location, Location[], or LocationLink[] (via `decodeRange` for generic JSON).
   - `Definition`; `References` (with `includeDeclaration: true`).
   - `Symbols`: flatten DocumentSymbol (with children into indent levels) or SymbolInformation, and map `SymbolKind` numbers to short kind names.
   - `HoverInfo{Signature, Doc, Empty}`, `Hover`, `parseHoverContents` (MarkupContent, MarkedString string or `{language, value}`, arrays), `splitMarkdown` (first fenced block becomes the code), `looksLikeDeclaration`, `stripMarkdown`, `highlightSnippet(code, rel)` using Chroma and `classFor`.
2. `calls.go`: `CallNode{name, detail, kind, path, line, col, sites, ext?, item}`, `treePath`, `callNode(raw, allow)`, `PrepareCalls`, `Calls(ctx, rel, item, outgoing)` (incoming uses `from`/`fromRanges`; outgoing uses `to`/`fromRanges`), `siteLines`, `firstSite`. `item` is the raw JSON string of the LSP item, echoed back by the client.
3. `lspsetup.go`:
   - `lspInstall.installsFor(goos)`, `registryFor(rel)`, `MissingLang(rel)`.
   - Types `lspSetupOption`/`Server`/`lspSetup`, `Setup(rel)`.
   - `lspJob{server, cmd, running, ok, error, output}`, `job(name)`, `Install(name, option)`: only a recipe with `Auto` for this OS whose tool (`Cmd[0]`) exists; reject otherwise; dedupe a running job.
   - `runInstall` (argv, output into a `tailBuffer` of about 8 KB, then Rescan on success), `tailBuffer`.
   - **`localPost`** exactly per §4.10.
   - Handlers `handleLSPSetup`, `handleLSPInstall`, `handleLSPStart`. Extend `lspBrief` with `missing`.
4. `server.go` handlers: `/api/lsp/def`, `/refs`, `/hover`, `/symbols`, `/calls`, `/warm`, `/setup`, `/install`, `/start`. Add `lspCtx` (wait default 10 s, max 120 s), `lspPos`, `lspRespond`. `resolvePath` now admits allowlisted external paths, so `/api/file` can open them.

## Tests
- Hover parsing: a MarkupContent with a go fence gives signature plus doc; a plain string; a `{language, value}` object; an array mixing both; empty content gives `Empty`.
- Location decoding: a single Location, an array, a LocationLink array (uses `targetSelectionRange`).
- With the fake server from step 09, extend it to answer `definition` with a location **outside** the root: the hit has `ext:true`, `Allowed` is true, and `/api/file?path=<abs>` succeeds while an unrelated absolute path gets 400.
- Extend the fake server for `prepareCallHierarchy` and `incomingCalls`: roots returned, expanding with `item` gives callers with `sites`.
- `localPost` table: GET → 405; Host `example.com:7777` → 403; Host `127.0.0.1:7777` with Origin `http://evil.com` → 403; Host `[::1]:7777` with a matching Origin → ok; `localhost` → ok.
- `Install`: an unknown server and a non-auto recipe index are rejected.

## Verify
```
go test ./... 2>&1 | tail -20
```
If gopls is installed: `curl 'localhost:7777/api/lsp/def?path=main.go&line=<n>&col=<c>'` on a stdlib call returns an `ext:true` hit.

Then follow the definition of done in copilot-instructions.md.
