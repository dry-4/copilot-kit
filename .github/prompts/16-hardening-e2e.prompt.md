---
mode: agent
description: "scry step 16: security audit, end-to-end HTTP tests, performance check"
---
# Step 16: Hardening, end-to-end tests, performance

Read `HANDOFF.md` first. No new features. Find and fix gaps.

## Read from SPEC.md
§2 (performance budgets), §7, §8, §11.

## Do
1. **Security audit.** Go through §8 item by item. For each item, grep the code, confirm it holds, and fix it if not. Specifically:
   - `grep -n 'exec.Command' *.go`: no `sh -c`, `bash -c`, or `cmd /c` with client-controlled input anywhere. The instruction text is a single argv element.
   - Every handler reading `path`, `dir`, or `glob` uses `safePath`/`resolvePath` (or the index's validated lookup).
   - Every process-running or writing handler calls `localPost` first.
   - Search snippets, file names, tree names, and hover docs are escaped in the frontend (grep `innerHTML` in `web/src` and check each use).
   - The Markdown sanitizer drops event handlers and `javascript:` URLs.

   Report a table: item, evidence (file:line), status.
2. **`scry_test.go` end-to-end** using `httptest.NewServer(NewServer(...))` over a temp workspace (a git repo with a few files, a `.gitignore`, an image, a binary, a Markdown file). Cover:
   - The happy path for every endpoint in §4.13 (LSP endpoints with no server: they return a `state` and no 500).
   - Path traversal: `../`, `..%2f`, `%2e%2e/`, absolute `/etc/passwd`, backslash `..\\` on all path-taking endpoints give 400 (or 404 for tree).
   - The external allowlist: not openable until allowed (call `lsp.allow` directly).
   - Gzip when `Accept-Encoding: gzip`; `Cache-Control: no-store`.
   - Methods: GET on POST-only endpoints gives 405.
   - Agent endpoints give 404 when disabled; mutations without Origin give 403.
   - `/api/search` and `/api/find` return `[]`, not `null`, when empty.
3. **Performance check.**
   - `git clone --depth 1 https://github.com/kubernetes/kubernetes /tmp/k8s`
   - Run `go build -o /tmp/scry . && /tmp/scry -no-open -port 7788 -no-telemetry /tmp/k8s`.
   - Measure: `indexMs` from `/api/meta`; `curl -w '%{time_total}'` for `/api/find?q=podspec` and `/api/search?q=PodSpec`; RSS after 20 s idle (`ps -o rss= -p <pid>`); opening a large generated file (e.g. `zz_generated*.go`) and fetching the last chunk.
   - Targets: index within a few hundred ms, fuzzy under 20 ms, search under 150 ms, idle RSS under 40 MB. If a target is missed by a lot, profile with a temporary `net/http/pprof` import (removed afterwards), fix the top hot spot, and re-measure. Report before and after numbers.
4. Run `go vet ./...` and `CGO_ENABLED=0 GOOS=windows go build ./...` (plus `freebsd` and `linux/arm`) to confirm cross-compilation still works.

## Verify
```
go test ./... 2>&1 | tail -30
```
Report the security table and the performance numbers in the final message and in `HANDOFF.md`.

Then follow the definition of done in copilot-instructions.md.
