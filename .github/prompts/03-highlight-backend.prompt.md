---
mode: agent
description: "scry step 03: windowed syntax highlighting backend + file endpoints"
---
# Step 03: Highlighting backend and file endpoints

Read `HANDOFF.md` first. Backend only in this step.

## Read from SPEC.md
§4.7 (all of it, carefully), §4.13 (rows `/api/file`, `/api/close`, `/api/raw`, `/api/metrics`), §4.14 (`metrics.go` only).

## Build
1. `go get github.com/alecthomas/chroma/v2`.
2. `highlight.go`, exactly per §4.7:
   - constants
   - `classFor` token mapping (use the table)
   - `htmlEscaper`, `visualWidth`
   - `Doc` with `lineOff`, `Lines`, `Exact`, `chunk` (context windows, byte cap, plain fallback), `backgroundPass`, `Raw`, `RawLines`
   - `highlightLines` (exactly `want` lines; tokens spanning newlines are split; empty class means bare escaped text)
   - `plainFallback`, `pad`
   - the LRU `hlCache` (`get`, `put`, `grow`, `evict` never removing the only remaining item, `remove` by prefix), `Evict`, `isBinary`, `Open` (64 MB cap, binary rejection, CRLF normalization, `path|mtime|size` key)
3. `metrics.go`: `getProcessMetrics()` returns `{rss, cpu}`. On Linux read `/proc/self/statm`. On macOS and BSD, current RSS isn't in `Getrusage` (that only reports the peak), so run `ps -o rss= -p <pid>`, cached for at least 2 s. CPU% comes from the delta of `syscall.Getrusage` user+system time between samples. Return zeros where unsupported. Must compile for all 15 targets in §6, so use build tags where needed.
4. `server.go`:
   - `/api/file` with the image detection map; `markdown:false`, `diffAvailable:false`, and `lsp:{"state":"off"}` for now; clamp `start`/`count`; `refine = !exact && coming`.
   - `/api/close` (Evict, then `debug.FreeOSMemory`).
   - `/api/raw` (`safePath` only, `http.ServeFile`, MIME type from the extension).
   - `/api/metrics`.
   - Include metrics in `/api/meta`.

## Tests (`highlight_test.go`, server tests)
- `Lines(0, n)` returns exactly n lines for n within, at, and past a chunk boundary.
- A Go file with a `/* ... */` block comment opening at line 995 and closing at 1010: after `backgroundPass`, the lines in chunk 1 inside the comment carry class `c`.
- A single 3 MB line gives plain escaped output, with no panic and a bounded time.
- Cache: with a small test budget (make `cacheBudget` a package var for the test), adding documents evicts the oldest and never the newest. `grow` accounting stays consistent. `Evict(abs)` removes all versions.
- `Open` rejects binary data (NUL byte), directories, and oversized files (use a temporary budget var or a sparse file).
- HTTP: `/api/file` returns `lines`, `total`, `lang`; an image returns `image:true`; `path=../x` returns 400; `/api/raw` serves bytes.

## Verify
```
go test ./... -run 'Highlight|Cache|Open|File' 2>&1 | tail -20
go run . -no-open -port 7777 . &  sleep 1
curl -s 'localhost:7777/api/file?path=main.go&start=0&count=5' | head -c 600; kill %1
```
Then follow the definition of done in copilot-instructions.md.
