---
mode: agent
description: "scry step 05: fuzzy finder, workspace search, outlines, regex definitions (backend)"
---
# Step 05: Fuzzy find, search, outline (backend)

Read `HANDOFF.md` first. Backend only in this step.

## Read from SPEC.md
§4.4, §4.5, §4.6, §4.13 (rows `/api/find`, `/api/search`, `/api/outline`, `/api/def`).

## Build
1. `fuzzy.go`: `FuzzyResult`, `isBoundary`, `fuzzyScore` (two-pass, exact weights from §4.4), and `FuzzyFind` (parallel chunks, sorted by score then path, empty query returns the first N).
2. `search.go`:
   - `Match`, `snip` (32/16/240 rules), `FileMatches`, `SearchOpts`
   - `searcher` (regex or word or case handling, ASCII-folded literal needle, glob via `compilePattern` from `ignore.go`)
   - `workBuf` with the 1 MB `release`, `asciiLower`, `readInto`, `scan` (whole-file `bytes.Contains` reject, per-line matches, `Def` classification)
   - `Search` (NumCPU workers, 8 MB skip, binary skip, `MaxFiles` stop, sorted, `truncated`)
   - `unindexedTarget`
3. `symbols.go`: the extension→family map, `declTemplates` and `declPatterns(ident)` (per extension; nil for non-identifiers), `Symbol{name, kind, line, indent}`, `outlineRules` for every family in §4.6, Markdown headings, the noise filter, and `Outline(abs, rel)`.
4. `server.go`:
   - `/api/find` (limit default 100, max 500)
   - `/api/search` (`q`, `re`, `case`, `word`, `glob`; 400 on a bad regex; `results` always `[]`)
   - `/api/outline`
   - `/api/def` (`lsp: {"state":"off","server":""}` for now; the LSP step fills it in)

## Tests
- fuzzy:
  - `srv` ranks `server.go` above `internal/some/deep/server_test_resolver.go`
  - `main` ranks `main.go` first
  - camelCase: `gd` matches `gotoDefinition.js` with positions on `g` and `D`
  - an empty query returns the first N
  - no match gives no result
- search:
  - literal case-insensitive and case-sensitive
  - regex with captures
  - whole-word excludes `foobar` for `foo`
  - glob `*.go` filters
  - binary file skipped
  - file over the cap skipped (use a small test var)
  - `MaxFiles` truncation sets `truncated`
  - snippet elision for a long indented line
  - `İ` in text does not misalign ASCII-folded matches
  - glob naming a gitignored file searches it
- symbols: outline cases for go, py, ts, rust, c, rb, and Markdown headings with indent; noise words excluded; `declPatterns("Foo")` matches `func Foo(`, `type Foo struct`, `class Foo`, `def Foo(` for their extensions and not `Foo()` calls.
- HTTP: `/api/def?sym=` returns declarations first and a `refCount`.

## Verify
```
go test ./... 2>&1 | tail -20
go run . -no-open -port 7777 . & sleep 1
curl -s 'localhost:7777/api/find?q=srv' | head -c 300; echo
curl -s 'localhost:7777/api/search?q=func&glob=*.go' | head -c 300; echo; kill %1
```
Then follow the definition of done in copilot-instructions.md.
