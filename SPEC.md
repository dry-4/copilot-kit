
# Task: Build "scry", a remote-first, read-optimized code viewer served from a single Go binary

You are a senior systems engineer. Build a complete, production-quality application from scratch in the current empty directory. Work autonomously in phases. After each phase, build, run the tests, and fix failures before moving on. Progress and cross-session state are kept in `HANDOFF.md`.

## 1. Product summary

scry is a code viewer. A user runs `scry` in a repository. It binds an HTTP server in under 1 ms, opens the browser, and serves a fast IDE-like reading interface: file tree, tabs, fuzzy file finder, workspace grep, symbol outline, syntax highlighting, git diffs, optional language-server navigation, and Markdown preview.

scry **has no text editor and never accepts file content from the client.** To change code, the user selects lines, types an instruction, and scry runs a coding-agent CLI already installed on the machine (`claude`, `gemini`, `cursor-agent`, or a custom command). When the CLI exits, scry detects what changed, reloads it, and offers a one-click undo of that edit.

Target uses: reviewing AI-agent output, browsing code on remote servers from a local browser, and reading huge repositories (the Linux kernel, ~96k files) on a laptop.

## 2. Non-negotiable constraints

1. **Language:** Go (use the current stable version; set the `go` directive in `go.mod` and CI to the *same* version). Module name `scry`, all code in `package main` at the repo root.
2. **Dependencies:** standard library, plus only `github.com/alecthomas/chroma/v2` (highlighting) and `github.com/yuin/goldmark` (Markdown). No CGO. Must build with `CGO_ENABLED=0`.
3. **Frontend:** vanilla JavaScript ES modules, no framework, no npm dependencies. Sources live in `web/src/*.js` and are bundled into `web/app.js` by `scripts/build-web.js` (use Bun if installed, otherwise a zero-dependency Node fallback). **Commit the bundled `web/app.js`** so `go build .` works without Node.
4. **Single binary:** embed `web/` with `//go:embed web`. Embed the `VERSION` file for the version string. A `-dev DIR` flag serves `DIR/web` from disk instead.
5. **Stateless in the workspace:** never create files or folders inside the workspace. Indexes, caches, and undo snapshots stay in memory. Per-user state goes in `$XDG_CONFIG_HOME/scry/` or else `~/.scry/`.
6. **Performance budgets:**
   - The listener is bound and the URL printed in under 1 ms. Indexing, git, LSP discovery, update checks, and browser launch all run in goroutines.
   - Indexing a 25k-file repo takes around 150 ms. Fuzzy find returns in single-digit milliseconds.
   - Opening a file costs O(window), not O(file length).
   - The idle process holds about 20 MB RSS.
7. **Fail quietly:** missing git, missing language servers, and missing harnesses must degrade features, never crash or return 500 for expected absences.
8. **Security:** the path sandbox and the `localPost` guard (see §8) are mandatory.

## 3. Repository layout to produce

```
main.go           CLI, target resolution, listen, browser, signals
ui.go             terminal output helpers; tty_linux.go, tty_bsd.go, tty_other.go (build-tagged isatty)
server.go         router, gzip, sandbox, scavenger, core handlers
index.go          workspace walk
ignore.go         gitignore engine
fuzzy.go          fuzzy finder
search.go         parallel grep
symbols.go        regex outlines + declaration patterns
highlight.go      windowed Chroma highlighting + LRU cache
markdown.go       goldmark rendering
git.go            git probe/status/diff/hunks
lsp.go            JSON-RPC stdio client
lspservers.go     registry, discovery, lifecycle
lspnav.go         definition/references/symbols/hover
calls.go          call hierarchy
lspsetup.go       install flow + localPost guard
agent.go          harness discovery + dispatch
agent_undo.go     pre-edit snapshot + undo
settings.go       settings.json
metrics.go        RSS/CPU sampling
update.go         self-update + daily check
telemetry.go      optional anonymous telemetry
*_test.go         tests
web/index.html, web/style.css, web/themes/*.css, web/src/*.js, web/app.js
scripts/build-web.js, Makefile, build.sh, install.sh, benchmark.sh
.github/workflows/release.yml
README.md, docs/internals/*.md, VERSION (start at 0.1.0), LICENSE (MIT)
```

## 4. Backend specification

### 4.1 CLI (`main.go`)
- Flags: `-port int=7777` (0 = OS picks), `-host string=127.0.0.1`, `-no-open`, `-no-lsp`, `-no-git`, `-dev string`, `-version`, `-v`, `-update`, `-no-color`, `-quiet`, `-no-telemetry`, `-agent string`, `-no-agent`. The positional argument `version` also prints the version. The usage header is `scry <ver> - a code navigator`.
- `-version` prints `scry <ver> (<GOOS>/<GOARCH>)`.
- `resolveTarget(target)`:
  1. Split a trailing `:line` or `:line:col` only if the literal path does not exist but the stripped path does. Handle Windows volume names.
  2. Resolve to absolute and evaluate symlinks.
  3. A directory becomes the root.
  4. For a regular file, the root is the git top-level if the file is inside a repo; otherwise the cwd if the target was relative and under the cwd; otherwise the parent directory. The initial file is the relative slash path.
- The viewer URL is `http://addr/?path=<rel>&line=<n>` (the query is omitted when empty).
- `listen`: port 0 goes straight to the OS. Otherwise try `port..port+99`, then fall back to the OS.
- Boot order: listen → `NewIndex` → `newLSPManager(root, !noLSP)` → telemetry → `NewServer` → `newAgentManager` (unless `-no-agent`; a bad `-agent` is fatal) → banner → `go openBrowser` → `go { ix.Build(); print "indexed N files  Xms"; list available LSP servers; telemetry session_started }` → `go checkDailyUpdate` → serve.
- Signals: the first SIGINT/SIGTERM prints "interrupted" and calls `Shutdown` with a 2 s timeout. A second signal calls `os.Exit(130)`. After `Serve` returns, close LSP and cancel the agent, then exit 130 if interrupted.
- Browser opener: try `$BROWSER` first. darwin: `open`. windows: `rundll32 url.dll,FileProtocolHandler`, then `cmd /c start`. Other: if WSL (env `WSL_DISTRO_NAME`/`WSL_INTEROP` or `/proc/version` contains microsoft/wsl), sleep 500 ms and try `wslview`, `powershell.exe Start-Process`, `cmd.exe /c start`; then `xdg-open`, `sensible-browser`, `gio open`, `google-chrome`, `firefox`, `chromium`. Use the first that starts.

### 4.2 Terminal UI (`ui.go`)
Helpers `uiHeading`, `uiKV(label, value, width)`, `uiStatus(role, msg, detail)` with glyph roles ok ✓ / warn ! / err ✗ / step › / info, plus `uiHint`, `uiBullet`, `uiAccent`, and `uiDim`. Use color only when the writer is a TTY (per-OS ioctl files with build tags) and `-no-color` is off. `-quiet` suppresses narration but not errors.

### 4.3 Index (`index.go`, `ignore.go`)
- `FileEntry{Path, Name, Size, lower, nameStart}`. `Node{Name, Path, Dir, Size, Ignored, Status, Dirty}` (JSON: `name, path, dir, size, ignored?, status?, dirty?`).
- `Index{root; mu RWMutex; files []FileEntry (sorted by path); children map[string][]Node; builtAt; buildMS; readyCh}`.
- `Build()`:
  - Start `gitStatus(root)` in a goroutine first.
  - Walk recursively with a semaphore of `NumCPU*4`; if the semaphore is full, recurse inline.
  - Per directory: read `.gitignore` and derive a child ignore set.
  - Skip symlinks. Ignored entries are appended with `Ignored:true` and never descended into (except `.git`/`.hg`/`.svn`, which are omitted entirely).
  - Sort nodes directories first, then case-insensitively. **Publish `children[""]` into the live index as soon as the root is read.**
  - After the walk, sort files, overlay git status on file nodes, set `Dirty` on all ancestors of changed paths, swap everything in under the lock, and close `readyCh` once.
- `Children(dir)`: return the indexed listing. If it is absent and `dir` lies under a directory listed as ignored (determined by the nearest visited ancestor; refuse `.`/`..`/backslash segments), read it from disk with every entry marked ignored.
- The gitignore engine supports comments, trailing-space trimming, `!` negation (last match wins), leading `/` anchoring, patterns containing `/` being anchored, trailing `/` meaning directory-only, `*`, `?`, `[...]`, and `**`. Use a literal fast path for patterns without wildcards and a regex otherwise. Expose `compilePattern(p) (rule, bool)` and `rule.hit(rel, isDir)` for reuse as the search glob filter.

### 4.4 Fuzzy finder (`fuzzy.go`)
- `FuzzyFind(files, query, limit) []FuzzyResult{path, name, pos []int}`. Lowercase the query and remove spaces. An empty query returns the first `limit` files.
- Score each file:
  1. Forward pass to confirm a subsequence match and record the end index.
  2. Backward pass from the end to collect the tightest positions, then reverse them.
  3. Score each position:
     - +12 if consecutive with the previous position; otherwise −min(gap, 12) (skipped for the first position)
     - +14 if the position is at or after the basename start
     - +16 if it is at index 0 or follows one of `/ _ - . space @`; otherwise +14 on a lower→Upper camel hump
     - +4 if the case matches exactly
  4. Adjust the total: −len(path)/8, −2 per `/`; +40 if the query appears verbatim in the lowercase basename, +20 more if at the basename start.
- Score in parallel across `NumCPU` chunks. Sort by score descending, then path. Endpoint limit: default 100, max 500.

### 4.5 Search (`search.go`)
- `SearchOpts{Query, Regex, Case, Word, Glob, MaxFiles=200, MaxPerFile=50, classifyDefs}`.
- Regex mode, or whole-word mode (`\b(?:q)\b` with QuoteMeta when not regex); add `(?i)` when case-insensitive. Otherwise use a literal needle, ASCII-folded when case-insensitive (document why: Unicode folding changes byte lengths).
- `NumCPU` workers, a jobs channel, and per-worker reusable read/fold buffers released when over 1 MB. Skip size 0, files over 8 MB, glob misses, and binary files (NUL in the first 8000 bytes). Stop reading new files once `MaxFiles` hits are reached.
- In literal mode, reject a file with `bytes.Contains` before splitting lines.
- `Match{line, pre, mid, post, def?}` via `snip`:
  - Trim leading whitespace from `pre`.
  - If `pre` is longer than 32 runes, keep "…" plus the last 16 runes.
  - Cap `mid` at 240 runes.
  - Give `post` whatever budget remains out of 240, ending with "…", and right-trim it.
- Sort results by path and set `truncated`.
- If `Glob` is a literal path (no wildcards, no `.`/`..` segments) naming an existing regular file that is **not** indexed, search just that file.

### 4.6 Outline and regex definitions (`symbols.go`)
- Map extensions to families: go; py; js (js, jsx, mjs, cjs, ts, tsx, svelte, vue); rust; jvm (java, kt, kts, scala, groovy, cs); c (c, h, cc, cpp, cxx, hpp, hh, m, mm); rb; php; sh; lua; elixir; swift (swift, dart); default.
- Per-family outline regex rules capture name and kind (kind may come from a capture group, such as struct or interface). Skip empty lines and lines over 500 characters, filter noise keywords (`if for while switch return else catch try do case with match defer go in`), and record the indentation. `.md`/`.markdown` produce `heading` symbols with indent = level−1.
- `declPatterns(ident)`: per-family "this line declares ident" regex templates, used to set `Match.Def`.
- `/api/def?sym=`: case-sensitive whole-word search with `MaxFiles` 400 and `MaxPerFile` 20. Keep only `Def` lines, deduplicated per path:line. Stable-sort files whose basename contains the symbol first. Return `{symbol, defs:[{path,...match}], refCount, lsp:{state,server}}`.

### 4.7 Highlighting (`highlight.go`)
- Constants: `maxFileBytes=64MB`, `hlChunk=1000`, `hlContext=400`, `hlWindowBytes=512KB`, `maxReportedCols=20000`, `bgLimit=2MB`, `cacheBudget=512MB`.
- `Open(abs, rel)`: stat; reject directories and files that are too big; cache key `abs|mtimeNanos|size`; read; reject binary files; normalize CRLF; build a `Doc` and put it in the cache.
- `Doc`:
  - Holds the source, `lineOff[]`, `Total`, `MaxCols` (tabs expand to 4-column stops; capped), a lexer (`lexers.Match(basename)`, then `lexers.Analyse`, then nil = "plain text"; wrap it with `chroma.Coalesce`), and a `chunks map[int][]string` guarded by an RWMutex with a `full` flag.
  - `Lines(start, end) ([]string, exact bool)` triggers a one-time background full pass when the source is at most `bgLimit`.
  - `chunk(c)`: lex lines `[c*1000 − 400, c*1000 + 1000 + 400]`. If that window exceeds 512 KB, drop the context. If it still exceeds 512 KB, use escaped plain text. Return the chunk and whether it came from the full pass.
  - The full pass replaces all chunks and sets `full=true`.
- `highlightLines(lexer, src, want)`: emit exactly `want` HTML lines. Each token becomes `<i class="X">escaped</i>`; tokens with an empty class are emitted as bare escaped text. Tokens spanning newlines are split across lines.
- Short classes by token type:

  | Token type | Class |
  | --- | --- |
  | KeywordType | `kt` |
  | Keyword (other) | `k` |
  | NameAttribute | `na` |
  | NameTag | `nt` |
  | NameDecorator | `nd` |
  | NameClass / NameNamespace / NameException | `nc` |
  | NameConstant | `no` |
  | NameProperty / NameLabel | `np` |
  | NameFunction | `nf` |
  | NameVariable | `nv` |
  | NameBuiltin | `nb` |
  | plain Name | none |
  | String | `s` |
  | Number | `m` |
  | Operator | `o` |
  | Punctuation | `p` |
  | CommentPreproc | `cp` |
  | Comment (other) | `c` |
  | GenericInserted | `gi` |
  | GenericDeleted | `gd` |
  | GenericHeading / GenericSubheading | `gh` |
  | GenericEmph | `ge` |
  | GenericStrong | `gs` |
  | Generic (other) | `g` |
  | Error | `err` |

- LRU cache (`container/list`): `get` / `put` / `grow(key, delta)` charge rendered chunk bytes (len + 16 per line). Evict from the back while over budget, never evicting the last remaining item. `Evict(abs)` removes all keys with the prefix `abs|`.

### 4.8 Markdown (`markdown.go`)
- goldmark with GFM, Footnote, auto heading IDs, `html.WithUnsafe()` (the client sanitizes).
- Custom heading ID generator that matches GitHub: lowercase; keep letters, digits, `-` and `_`; spaces become `-`; drop everything else; append `-1`, `-2` for duplicates.
- AST transformer adding `data-line="<source line>"` to block nodes.
- Custom fenced-code renderer that highlights with Chroma using the same classes (plain text above 256 KB).
- Refuse files over 4 MB. `GET /api/markdown?path=` → `{path, html}`.

### 4.9 Git (`git.go`)
- `gitDisabled` global (`-no-git`). `gitProbe(root)`: LookPath git + `git -C root rev-parse --show-toplevel`, memoized.
- `gitStatus(root) map[rel]letter`: `git -C root status --porcelain=v2 -z --untracked-files=all` (or equivalent). Parse records `1` (ordinary), `2` (rename: consumes an extra NUL-separated original path), `u` (unmerged → `!`), and `?` (untracked → `?`). Map XY codes by preferring the index side unless it is `.`: A, D, R, C → the same letter; U → `!`; everything else → `M`. Convert repo-relative paths to paths relative to the served root (which may be a subdirectory) and drop paths outside it. Return nil when empty or on error.
- `gitDiff(root, rel)`: `git -C root diff --no-color HEAD -- rel`, returning `""` on error.
- `gitHunks`: walk the diff. For each maximal run of +/- lines inside hunks: deletions and additions together → `modified` lines; additions only → `added`; deletions only → `deleted` marker at the (new line − 1) where the run began. Take the new start from `@@ -a,b +c,d @@`. Skip `\ No newline` lines.

### 4.10 LSP (`lsp.go`, `lspservers.go`, `lspnav.go`, `calls.go`, `lspsetup.go`)
- **Registry** (ordered; the first installed server claims each extension):

  | Name | Command | Extensions | Default languageId | Install recipes |
  | --- | --- | --- | --- | --- |
  | gopls | `gopls` | `.go` | `go` | `go install golang.org/x/tools/gopls@latest` (auto), brew (darwin, auto) |
  | rust-analyzer | `rust-analyzer` | `.rs` | `rust` | `rustup component add rust-analyzer` (auto), brew (darwin) |
  | pyright | `pyright-langserver --stdio` | `.py`, `.pyi` | `python` | `npm install -g pyright`, brew |
  | pylsp | `pylsp` | `.py`, `.pyi` | `python` | `pipx install python-lsp-server`, brew |
  | ruff | `ruff server` | `.py` | `python` | none offered |
  | typescript | `typescript-language-server --stdio` | `.ts .tsx .js .jsx .mjs .cjs` | `javascript` (`.ts`→`typescript`, `.tsx`→`typescriptreact`, `.jsx`→`javascriptreact`) | `npm install -g typescript-language-server typescript`, brew |
  | clangd | `clangd --background-index` | `.c .h .cc .cpp .cxx .hpp .hh .m .mm` | `cpp` (`.c`/`.h`→`c`) | brew llvm (darwin, auto); `sudo apt install clangd` (linux, display only); `winget install LLVM.LLVM` (windows, display only) |
  | zls | `zls` | `.zig` | `zig` | brew |
  | lua | `lua-language-server` | `.lua` | `lua` | brew |
  | solargraph | `solargraph stdio` | `.rb` | `ruby` | `gem install solargraph`, brew |
  | jdtls | `jdtls` | `.java` | `java` | brew |
  | omnisharp | `omnisharp -lsp` | `.cs` | `csharp` | none |
  | texlab | `texlab` | `.tex` | `latex` | brew |

- **`lspBinDirs()`:** `$GOBIN`, each `$GOPATH/bin`, `~/go/bin`, `~/.cargo/bin`, `~/.local/bin`, the directory of `npm`; darwin: `/opt/homebrew/bin`, `/usr/local/bin`, `/opt/homebrew/opt/llvm/bin`, `/usr/local/opt/llvm/bin`; windows: `%APPDATA%\npm`, `%ProgramFiles%\LLVM\bin`. `lookPathIn(name, dirs)` checks PATH first. `discover()` runs in the background and swaps in the result atomically. `Rescan()` repeats it and clears failures.
- **Client:**
  - Content-Length framed JSON-RPC 2.0 over the child's stdin/stdout, with a read loop dispatching responses by id.
  - Reply `null` to server→client requests (`workspace/configuration` gets an array of nulls, and so on).
  - Track `$/progress` begin/end tokens to compute `busy()`.
  - `initialize` with rootUri, workspaceFolders, and capabilities (hover markdown, definition linkSupport, references, documentSymbol hierarchical, callHierarchy, workDoneProgress), then `initialized`.
  - `ensureOpen(abs)` sends `didOpen` once with the file text. `closeDoc` sends `didClose`.
  - Positions convert between byte columns and UTF-16 code units in both directions.
  - `shutdown` sends shutdown/exit, then kills the process after a timeout.
  - Discard stderr.
- **Manager:**
  - `client(ctx, rel)`: return a live client. If the client is dead, respawn it (max 3 restarts per server). If a start is in flight, wait on its channel. Otherwise spawn with a 30 s handshake and record failures.
  - `State(rel)` → off | starting | indexing | ready | failed, plus the server name. While discovery hasn't finished and a registry entry exists, report `starting`.
  - `Close()` shuts everything down.
- **Navigation:**
  - `Definition` / `References` (includeDeclaration) accept Location, Location[], or LocationLink[] and return `NavHit{path, line, col, endCol, text(snippet), ext?, def?}`.
  - A target outside the root is added to the `external` allowlist (max 20k) and returned with an absolute path and `ext:true`.
  - `Symbols`: flatten DocumentSymbol or SymbolInformation into `Symbol{name, kind, line, indent}`.
  - `Hover`: parse MarkupContent, MarkedString, or an array; split into the first code block (signature, highlighted via Chroma for the file's language) and prose (doc); `looksLikeDeclaration` heuristics.
- **Call trails:**
  - `PrepareCalls(abs, rel, line, col)` → `textDocument/prepareCallHierarchy` → `CallNode{name, detail, kind, path, line, col, sites[], ext?, item(opaque JSON string)}`.
  - `Calls(rel, item, outgoing)` → `callHierarchy/incomingCalls` or `outgoingCalls`, with `sites` taken from fromRanges.
- **Setup:**
  - `Setup(rel)` returns `{lang, installed, servers:[{name, options:[{cmd, auto, os}]}], job}` for the file's language.
  - `Install(server, option)` runs only registry recipes marked `auto` for the current OS. It streams output into a tail buffer (a few KB), records `running`, `ok`, and `error`, rescans when done, and dedupes concurrent jobs per server.
- **`localPost(w, r)`** (used by every mutating endpoint):
  1. The method must be POST, else 405.
  2. The Host with the port stripped and brackets trimmed must be `localhost` or an IP literal, else 403 with the message "open scry by IP address or localhost…".
  3. The parsed `Origin` host must equal `r.Host`, else 403 "request did not come from scry".

### 4.11 Agent editing (`agent.go`, `settings.go`)
- **Presets:**
  - `claude`: `claude -p --permission-mode acceptEdits {prompt}`
  - `gemini`: `gemini --approval-mode auto_edit -p {prompt}`
  - `cursor-agent`: `cursor-agent -p --force {prompt}`
- **`resolveAgentSpec(spec)`:** a case-insensitive preset name, or a whitespace-split template that must contain `{prompt}` (the error lists the known presets). The binary must be found via `lookPathIn` with `lspBinDirs`. Returns (display name, argv with the resolved binary).
- **`agentManager{root, lsp, mu, selected, args, pinned, job, undo, cancel, seq}`:**
  - `-agent` pins the harness for the session; select then returns an error.
  - Otherwise restore the harness from `settings.json` `{"agent": spec}`, silently ignoring a stale value.
  - `Select("")` clears the choice and writes empty settings. `Select(name)` persists the *spec as given*.
  - `Detect()` rescans presets on every call and returns `[{name, cmd, installed, path}]`.
- **`Start(abs, rel, l1, l2, instruction, force)`:**
  - An empty instruction is an error. No harness → `errAgentNone`. A running job → `errAgentBusy` (409).
  - If git is available and `rel` is in git status and `!force` → `errAgentDirty` (409, message contains "uncommitted").
  - Read lines l1..l2 (1-based, clamped).
  - Increment seq, clear `undo`, create the job, and launch `run` in a goroutine. Return a snapshot.
- **Prompt:**
  ````
  ### Reference: <rel>:<l1>[-<l2>]
  ```<ext>
  <snippet>
  ```

  ### Instruction
  <instruction>

  Edit the file in place to carry out that instruction. Change only what it asks for, and do not explain the change afterwards.
  ````
- **`run`:**
  1. Use a context with a 10-minute timeout, stored as `cancel`.
  2. `pre := capturePreEdit(root)`.
  3. Substitute `{prompt}` in each argv token. Run with `cmd.Dir = root`, stdout and stderr into 32 KB tail buffers, and empty stdin. A timeout becomes the error "gave up after 10m0s".
  4. `changed := changedSinceMaps(pre.status, worktreeSnapshot(root))`, where `worktreeSnapshot` is git status with `" size mtimeNanos"` appended per existing path, and the comparison runs in both directions.
  5. `settle(changed)`: `Evict` plus `lsp.CloseDoc` for each changed file.
  6. If anything changed, `planUndo`.
  7. Finalize the job: `Running=false, Changed, Ms, Undoable, UndoNote, Error`. `Tracked = gitAvailable`.
  8. Log the outcome to the terminal.
- **Job snapshot JSON:** `{id, harness, path, lines, running, error?, log, stdout?, stderr?, changed[], ms, undoable, undoNote?, tracked}`.
- **Cancel:** kill via the context. Written changes stay.
- **Settings path:** `$XDG_CONFIG_HOME/scry/settings.json`, else `~/.scry/settings.json`. `readSettings` never fails. Writes are mutex-guarded, create directories, and write indented JSON with a trailing newline.

### 4.12 Undo (`agent_undo.go`)
- **`capturePreEdit`:** status snapshot; `HEAD` via `git rev-parse --verify -q HEAD`. For each status path: skip trailing `/` (untracked directories); if missing on disk, set `missing[rel]`; if it is a regular file ≤ 8 MB and the running total ≤ 64 MB, save its bytes and permission bits.
- **`planUndo(root, pre, changed)`** is all or nothing. It returns `(*agentUndo, reason)`. For each changed path:
  - A directory path (trailing `/`): if it was already in status, abort with the reason "files were added to the untracked directory X"; otherwise remove it.
  - Missing before: remove.
  - Dirty before: restore the saved copy, or abort with "X was too large to keep a copy of".
  - Clean before: fetch the HEAD blob with `git -C root ls-tree -z HEAD -- rel` (empty → remove; `100755` → 0755; `100644` → 0644; anything else is an error) and `git -C root cat-file blob HEAD:./rel`.
  - Record `stamps[rel]` = `"size mtimeNanos"`, or `"-"` if absent.
- **`Undo(force)`:**
  - Busy → 409. No plan → 404 "there is no agent edit to undo".
  - Unless forced, any stamp mismatch → 409 "<paths> changed since the agent edit".
  - Consume the plan (`undo = nil`, `job.Undoable = false`) before applying it.
  - Apply: `RemoveAll`, or `restoreFile`, which creates parent directories, removes any non-regular file in the way, writes the data, and chmods.
  - `settle`. Partial failure → 500 listing the failures.
  - Success → `{undone: paths}`.

### 4.13 Server (`server.go`)
- `ServeHTTP` wrapper: store `lastReq` atomically; set `Cache-Control: no-store`; if the client accepts gzip, wrap the writer with a `sync.Pool` of `gzip.BestSpeed` writers and set `Content-Encoding` and `Vary`.
- `scavenge()`: tick every 10 s. After 15 s idle, following activity, call `debug.FreeOSMemory()` once.
- `writeJSON` uses `SetEscapeHTML(false)`. `fail(w, code, msg)` → `{"error": msg}`.
- `safePath(rel)`: trim spaces and a leading `/`, then `filepath.Clean`. `.` means the root. Reject `..`, `../…`, absolute paths, and anything not under the root. Returns `(abs, relSlash, ok)`.
- `resolvePath(p)`: an absolute path is allowed only if it is in the LSP allowlist; otherwise use `safePath`.
- **Endpoints** (use `resolvePath` unless noted):

| Path | Behavior |
| --- | --- |
| `GET /` | `web/index.html` (404 for other unknown paths) |
| `/static/*` | file server over the `web/` subtree |
| `/static/themes.css` | concatenation of `web/themes/*.css` in name order with `/* themes/x.css */` headers |
| `/api/meta` | `{root, name, files, indexMs, builtAt, ready, git, lspServers, metrics, version, agent, agentPinned, agents}` |
| `/api/metrics` | `{rss, cpu, ...}` |
| `/api/tree?dir=` | `{dir, children}`; while not ready, poll 30×10 ms; 404 if unknown |
| `/api/find?q=&limit=` | `{results}` |
| `/api/file?path=&start=&count=` | images → `{path, image:true, size}`; otherwise `{path, lang, total, maxCols, start, lines, size, exact, refine: !exact && bgComing, markdown, diffAvailable, lsp: brief}`; 415 for binary/too large |
| `/api/close?path=` | Evict, LSP CloseDoc, FreeOSMemory → `{ok, path}` |
| `/api/raw?path=` | `safePath` only; `http.ServeFile` with the mime type from the extension |
| `/api/markdown?path=` | see 4.8 |
| `/api/diff?path=` | `{path, diff, available}` |
| `/api/gutter?path=` | `{path, available, added, modified, deleted}` (arrays never null) |
| `/api/search?q=&re=1&case=1&word=1&glob=` | `{results, files, total, truncated}`; 400 on a bad regex |
| `/api/outline?path=` | `{path, symbols}` |
| `/api/def?sym=&path=` | see 4.6 |
| `POST /api/reindex` | rebuild → `{files, indexMs}` (make it POST-only; update the client accordingly) |
| `/api/lsp/def`, `/api/lsp/refs` | `path, line (1-based), col (UTF-16), wait (ms, default 10000, max 120000)` → `{hits, state, server, error?}` |
| `/api/lsp/hover` | → `{signature, doc, empty, state, server}` or `{empty:true, state, server, error}` |
| `/api/lsp/symbols` | → `{symbols, state, server, error?}` |
| `/api/lsp/calls` | position → roots; `item` + `dir=out|in` → children → `{nodes, state, server, error?}` |
| `/api/lsp/warm?path=&wait=` | start the client with a short wait (default 1 ms, max 60 s) → brief `{state, server, missing?}` |
| `/api/lsp/setup?path=` | setup info |
| `POST /api/lsp/install?server=&option=` | localPost → job |
| `POST /api/lsp/start?path=` | localPost → Rescan, start with a 1.5 s wait → brief |
| `/api/agent/harnesses` | `{harnesses, selected, pinned, settings}` |
| `POST /api/agent/select?name=` | localPost; 409 if busy |
| `POST /api/agent/edit?path=&l1=&l2=&instruction=&force=1` | localPost; 409 if busy or dirty |
| `/api/agent/job` | snapshot or `{idle:true}` |
| `POST /api/agent/cancel` | localPost → `{cancelled}` |
| `POST /api/agent/undo?force=1` | localPost; 404/409/500 as specified |

All `/api/agent/*` endpoints return 404 "editing is not available in this session" when the agent is disabled.

### 4.14 Metrics, update, telemetry
- **`metrics.go`:** RSS from `/proc/self/statm` on Linux or `getrusage`/`ps` fallbacks elsewhere; CPU% sampled from the delta of process CPU time. Must work (or return zeros) on every target OS.
- **`update.go`:**
  - The repo defaults to `scry-app/scry` (override with `SCRY_REPO`).
  - State lives in `~/.scry/update_state.json` as `{last_checked, latest_version}`.
  - `checkDailyUpdate` skips when quiet, checks at most every 24 h via the GitHub `releases/latest` API with a short timeout, and prints a notice to stderr.
  - `runSelfUpdate`:
    1. Find the asset `scry-<ver>-<goos>-<goarch>[.exe]`. **Require** `checksums.txt` (sha256sum format) and verify it.
    2. Write a temp file next to the executable, falling back to the OS temp directory. Suggest sudo on permission errors.
    3. chmod 755 and run `-version` as a smoke test.
    4. Replace the binary with rename. On a cross-device error, copy instead. On Windows, move the current binary to `.old` first and roll back on failure.
  - Implement `compareSemver`.
- **`telemetry.go`:**
  - Enabled only when a key is present (ldflags `-X main.posthogKey=` or `SCRY_POSTHOG_KEY`) and the user hasn't opted out (`-no-telemetry`, `DO_NOT_TRACK=1`, `SCRY_TELEMETRY in {0,false,off,no}`).
  - Host from `SCRY_POSTHOG_HOST`, default `https://us.i.posthog.com`, POST `/capture/`.
  - Anonymous ID in `~/.scry/anonymous_id`; session UUID v4.
  - Events: `session_started` with `{files_bucket, index_ms, has_git, has_lsp}`; `session_ended` and `session_stopped` with `{duration_seconds, exit_reason}`, sent synchronously on close.
  - Queue of 128 with a non-blocking enqueue, a background worker, a 4 s client timeout, and `SCRY_TELEMETRY_DEBUG=1` logging.
  - **Never send paths, file names, code, queries, or symbols.**

## 5. Frontend specification

### 5.1 Structure (`web/index.html`)
One page with these regions and IDs:

- `#rail`: activity rail (Explorer, Search, Outline, Theme, Shortcuts)
- `#side`: sidebar with `#panel-files`, `#panel-search`, `#panel-outline`
  - header: `#root-name`, `#btn-changed` (hidden unless git), `#btn-reindex`
  - footer: logo, `#st-ver`, GitHub link, `#btn-theme`
- `#resizer`, `#tabs`, `#crumbs`
- `#editor` > `#viewport` > `#sizer` + `#rows`
- `#mdview` > `article#md.md`, with a `#md-switch` Preview/Source toggle
- `#diffview`, `#empty` (welcome screen with `#empty-ver`), `#hovercard`, `#findbar`
- `#agentbox`: `#agent-ref`, `#agent-harness`, `#agent-pick`, `#agent-input`, `#agent-err`
- `#sel-menu`, `#agent-menu`, `#toast`
- `footer#status`: language, lines, size, cursor, LSP state, index time, memory; `#footer-sel` selection actions; `#footer-actions` with the `Agent:` switch and Undo Edit
- `#overlay` > `#palette`, `#helpsheet`
- Right-side inspector panel with its own resizer

### 5.2 Modules (`web/src/`) and responsibilities
- **`state.js`:** global state `S` (docs/tabs, active index, wrap, lineNumbers, mdPreview, meta, lsp state), `$`/`$$`, `esc`, `api(path, params)` (GET; throws when `j.error`), `apiPost`, `debounce`, `isMac`, `MOD`, constants `CHUNK=1000`, `LH`, `OVERSCAN`. Shortcut labels: `keyLabel("Mod+Shift+F")` renders `⌘⇧F` on Mac and `Ctrl+Shift+F` elsewhere; the `"pc|mac"` combo syntax; `applyKeyLabels()` fills `[data-keys]` elements.
- **`renderer.js`:** measure line height and character width; `layout()` sets the sizer height/width from `total` and `maxCols` (wrap-aware); `render()`/`paint()` mount only rows in `[scrollTop/LH − OVERSCAN, … + OVERSCAN]`; `ensureChunks` fetches missing 1,000-line chunks; `refineChunk` re-fetches chunks marked `refine` after 800 ms with bounded retries; decorations (find hits, occurrences, gutter markers, selection); caret; `toPos`/`toPoint` DOM↔(line,col); toggles for wrap and line numbers.
- **`tabs.js`:** `openFile(path, {line, col})` fetches the first chunk, creates or reuses a tab, pushes history, loads the gutter, warms the LSP, and handles images; `closeTab` calls `/api/close`; `reopenClosedTab`; `switchTab`; `drawTabs`; `drawCrumbs`; `reloadOpenTabs()` refreshes every open tab in place, keeping scroll and view mode.
- **`tree.js`:** lazy directory expansion via `/api/tree`; git status badges; dirty dots; dimmed ignored entries; changed-only filter; `revealFile`.
- **`palette.js`:** one overlay with modes: `>` commands, `@` symbols in the current file, `:` line, default files (`/api/find` with highlighted `pos`). Keyboard navigation. A `COMMANDS` list for every action.
- **`search.js`:** query box with regex/case/word toggles and a glob; grouped results; clicking opens at the line.
- **`outline.js`:** regex outline immediately, upgraded to LSP symbols when ready; kind badges.
- **`find.js`:** in-file find seeded with the selection, next/previous, minimap dots.
- **`cursor.js`:** click, keyboard caret movement (Left/Right/Home/End, Cmd+arrows on Mac), word at point, occurrence highlighting.
- **`hover.js`:** debounced hover → `/api/lsp/hover` → card with a highlighted signature, doc, and Definition/References buttons; Cmd/Ctrl-hover underlines identifiers as links.
- **`lsp.js`:** `gotoDefinition` (LSP, falling back to `/api/def`; a single hit jumps, several show a picker), `findReferences` (to the inspector), `warmLSP`, status updates.
- **`inspector.js` / `calls.js`:** right panel tabs for References and Call Trail; a lazily expandable callers/callees tree; click to jump.
- **`lspsetup.js`:** when state is off and a language is missing, offer install options (run auto recipes, show others with copy buttons), poll the job, then start.
- **`diff.js`:** parse unified diffs into hunks; render split (paired rows) or unified tables with line numbers and markers; `Cmd+D` toggles; remember the layout in localStorage.
- **`markdown.js`:** fetch `/api/markdown`; **sanitize** by parsing into a template and rebuilding only allowlisted tags and attributes (drop scripts, event handlers, and `javascript:` URLs); resolve relative links (in-workspace `.md` opens in scry; anchors scroll) and images (through `/api/raw`); sync scroll between preview and source using `data-line`; `Alt+M` toggle.
- **`selbar.js`:** while a selection exists, the footer shows Copy Ref (`path:l1-l2`), Copy for Agent (the same markdown shape as the agent prompt), Edit with Agent, and Find Usages; right-click opens `#sel-menu` built from those buttons; Select All in the code view; in the diff view, `diffSelection()` maps the selection (either side, including deleted lines) to working-tree line numbers.
- **`agent.js`:**
  - `applyAgentMeta` shows the footer `Agent: <name>`, or `Agent: choose`, or hides it when unavailable.
  - `openAgentEdit({path, l1, l2, fromDiff})` shows the composer; if no harness is selected, it shows the picker first (installed harnesses selectable, others listed with their commands and the settings path).
  - Enter submits via `apiPost('/api/agent/edit')`. On a 409 with "uncommitted", confirm, then retry with `force=1`. `fromDiff` always forces.
  - Poll `/api/agent/job` about every 700 ms while running and show elapsed time. On an error, show the error plus stdout/stderr blocks inline and keep the input.
  - On success: POST `/api/reindex`, `reloadOpenTabs()`, redraw the tree, toast "Changed N files" (or reload everything when `tracked` is false), and show Undo Edit when `undoable` (with `undoNote` as a tooltip otherwise).
  - Undo calls `apiPost('/api/agent/undo')`; on a 409 stale response, confirm and force; then reload the same way.
  - The footer harness menu switches among installed harnesses.
- **`status.js`:** status bar fields; `setLspState`; memory display that refreshes periodically; `fitStatus()` adds `fit-1..fit-6` classes to progressively hide detail so the bar never wraps.
- **`panels.js`:** sidebar panel switching, the reindex button, the draggable resizer (persisted).
- **`history.js`:** back/forward stack of `{path, line}`.
- **`shortcuts.js`:** global key handler for every shortcut in §5.3 and the `?` help sheet generated from one table.
- **`theme.js`:** discover `[data-theme="…"]` selectors from the loaded stylesheet, set/cycle the theme, persist it in localStorage.
- **`main.js`:** init all modules; boot: restore prefs (wrap, lines, mdPreview default true), `applyKeyLabels`, measure, `/api/meta`, draw the tree root, open `?path=&line=` then strip it from the URL, re-measure after `document.fonts.ready`, poll meta every 150 ms until `ready`.

### 5.3 Keyboard shortcuts (implement all; `Mod` = Cmd on macOS, Ctrl elsewhere)

| Shortcut | Action |
| --- | --- |
| `Mod+K` | Universal palette |
| `Mod+P` | Go to file |
| `Mod+Shift+P` | Command palette |
| `Mod+Shift+O` | Go to symbol in file |
| `Mod+Shift+F` | Workspace search |
| `Mod+F` | Find in file |
| `Mod+G` | Jump to line |
| `Mod+D` | Toggle diff |
| `F12` / `Mod+Click` | Go to definition |
| `Shift+F12` | Find references |
| `Alt+Shift+H` | Call trail |
| `Alt+Z` / `Alt+L` / `Alt+M` | Toggle word wrap / line numbers / Markdown preview |
| `Alt+C` / `Alt+A` / `Alt+U` / `Alt+E` | With a selection: copy ref / copy for agent / find usages / edit with agent |
| `Alt+Left` / `Alt+Right` | History back / forward |
| `Mod+B` | Toggle sidebar |
| `Alt+W` (and `Mod+W` where allowed) | Close tab |
| `Alt+Shift+T` | Reopen closed tab |
| `Ctrl+Tab` | Next tab |
| `Alt+1..9` | Select tab by position |
| `?` | Shortcut help sheet |
| `Esc` | Close overlays |

### 5.4 Styling
- `style.css` uses CSS custom properties only for colors (`--bg`, `--fg`, `--muted`, `--accent`, `--border`, `--sel`, `--gutter-add`/`mod`/`del`, token colors `--t-k`, `--t-s`, `--t-c`, …).
- Monospace code font, a dense layout, a 1-line-height grid for the virtual rows.
- Map token classes (`.k`, `.kt`, `.s`, `.c`, …) to the token variables.
- Ship 14 themes in `web/themes/` as `[data-theme="name"]{...}` blocks: `dark` (default, "Tokyo Night"-like), `light`, `catppuccin-mocha`, `catppuccin-latte`, `dracula`, `github-dark`, `gruvbox-dark`, `gruvbox-light`, `monokai`, `nord`, `one-dark`, `rose-pine`, `solarized-dark`, `solarized-light`.

## 6. Build, packaging, release
- **`scripts/build-web.js`:** if `bun --version` works, run `bun build web/src/main.js --outfile=web/app.js --target=browser --format=iife`. Otherwise use the Node fallback:
  1. Walk relative imports depth-first from `main.js`.
  2. Strip `import` lines and `export` keywords.
  3. Concatenate modules in dependency order inside `(() => { 'use strict'; ... })();`.
  4. **Fail** if any top-level name is declared in two modules.
  5. Print the size and time.
- **Makefile targets:**
  - `build` (web + `go build -trimpath -ldflags "-s -w [-X main.posthogKey=$POSTHOG_KEY]" -o scry .`)
  - `web`
  - `test` (web + `go test -v ./...`)
  - `dist` (`./build.sh`)
  - `publish <version>` (write VERSION, build dist, commit "Release vX", tag -fa, print the push command)
  - `clean`, `help`
- **`build.sh`:** `CGO_ENABLED=0` cross-compile to linux/{amd64,arm64,arm,386,riscv64}, darwin/{amd64,arm64}, windows/{amd64,arm64,386}, freebsd/{amd64,arm64}, openbsd/{amd64,arm64}, netbsd/amd64 as `dist/scry-<ver>-<os>-<arch>[.exe]`.
- **`.github/workflows/release.yml`:** on `v*` tags or manual dispatch: checkout, setup Go (same version as go.mod), setup Node 20, bundle, write VERSION from the tag, cross-compile (optional `POSTHOG_KEY` secret), `sha256sum scry-* > checksums.txt`, create a GitHub release with `softprops/action-gh-release@v2`.
- **`install.sh`:** POSIX sh:
  1. Detect OS (linux/darwin/freebsd/openbsd/netbsd/windows via msys) and arch (amd64/arm64/arm/386/riscv64).
  2. Resolve `VERSION` (`latest` via the GitHub API).
  3. Download the asset and `checksums.txt`, and verify with sha256sum/shasum.
  4. Install to `INSTALL_DIR` (default `/usr/local/bin` if writable, else `~/.local/bin`, with a PATH hint).
  5. `SCRY_REPO` override. Colored ✓ › ! ✗ log lines.
- **`benchmark.sh`:** `--clone` (shallow-clone flask, redis, react, django, kubernetes, TypeScript, linux into `bench-repos/`), default mode (start scry with `-no-open -port 0` per repo and measure index time from `/api/meta`, fuzzy and search latency via curl timing, RSS via ps), `--vscode DIR` (compare RSS/CPU/process count against running VS Code server processes), `--memory DIR` (RSS over index → search burst → 20 s idle), `--lsp DIR` (definition/hover/references latency). Output markdown tables.

## 7. Testing requirements
Write table-driven Go tests. At minimum:

- **Target resolution:** directories, files in and out of git, `file:42`, `file:42:7`, and paths with real colons.
- **`listen`:** port walking.
- **Ignore engine:** negation, anchoring, `**`, directory-only rules, nested `.gitignore` files.
- **Index:** ignored entries listed but not indexed, `.git` hidden, symlinks skipped, dirty ancestors, browsing inside ignored directories, rejection of crafted `..` paths.
- **Fuzzy:** basename beats a deep path; exact prefix ranks first.
- **Search:** literal/regex/word/case modes, glob filtering, binary skip, truncation, snippet elision, the unindexed single-file glob, and ASCII folding with non-ASCII text.
- **Highlight:**
  - windows produce exactly `want` lines
  - a block comment opening before a chunk boundary is highlighted correctly after the full pass
  - huge single-line files fall back to plain text
  - the cache budget evicts old entries but never the newest
  - `Evict` works
- **Git:** status parsing of porcelain v2 records (including renames and a served subdirectory), and hunk classification into added/modified/deleted. Use temporary repos created with `git init` in tests; skip if git is missing.
- **LSP:** frame read/write, UTF-16 conversion with emoji and CJK, hover content parsing variants, registry precedence, and a fake stdio server (the test binary re-executing itself as a helper) to exercise initialize, definition, and crash-restart.
- **`localPost`:** wrong method, hostname Host, mismatched Origin, and the valid case.
- **Agent:**
  - spec resolution (presets, templates without `{prompt}`)
  - prompt format
  - dirty guard and force
  - single flight
  - change detection including an already-modified file edited again
  - undo for dirty/clean/new files and new directories
  - stale refusal and force
  - size-cap refusal
  - undo used once

  Use a fake harness shell script passed as a template.
- **Markdown:** GitHub heading IDs (including duplicates and Unicode), `data-line` attributes, highlighted fences.
- **Update:** semver comparison, checksum parsing and mismatch rejection (using httptest).
- **Telemetry:** opt-out environment handling, file-count buckets, no network when disabled.
- **End-to-end server tests** (`httptest`): every endpoint's happy path and error codes, path traversal attempts (`../`, encoded, absolute) are rejected, the LSP external allowlist, gzip, and agent endpoints returning 404 when disabled.

## 8. Security checklist (verify before finishing)
- [ ] No endpoint writes client-supplied content to disk.
- [ ] Every path parameter goes through `safePath`/`resolvePath`.
- [ ] Every endpoint that executes a process or writes files uses `localPost`.
- [ ] Commands executed come only from the hard-coded registry, the presets, or the user's `-agent`/saved template. The client supplies only the instruction text, which is passed as a single argv element (never through a shell).
- [ ] Markdown HTML is sanitized client-side before insertion. Search snippets and file names are escaped.
- [ ] The README warns that binding `-host 0.0.0.0` lets anyone who can reach the port by IP run the chosen harness.

## 9. Documentation to write
- **`README.md`:** pitch, install (curl, from source), features, LSP table, agent editing (how an edit works, failures, undo, guards), usage examples, remote usage, flags table, shortcuts table, philosophy, telemetry and privacy (what is and is never collected; how to opt out), benchmarks section, contributing, license.
- **`docs/internals/`:** architecture (startup, API catalog, memory, security), indexing-and-ignore, fuzzy-search, workspace-search, syntax-highlighting, editor-virtualization, markdown, git-integration, lsp-and-intelligence, agent-editing, file-reload-and-updates, styling-and-themes.
- **`CONTRIBUTING.md`, `PUBLISHING.md`, `AGENT.md`:** rules for AI contributors, including keeping docs in sync with code.

## 10. Phased execution plan
Complete each phase fully (code + tests passing + `go vet` clean) and update `BUILD_PROGRESS.md` before starting the next.

1. **Skeleton:** go.mod, VERSION, main.go flags/target/listen/browser/signals, ui.go, tty files, server.go with embed, gzip, sandbox, scavenger, `/`, `/static`, `/api/meta`; a minimal index.html. Verify: `go run . .` serves the page.
2. **Index + tree:** ignore.go, index.go, `/api/tree`, `/api/reindex`; frontend state.js, ui.js, tree.js, panels.js, main.js boot, build-web.js.
3. **File viewing:** highlight.go, `/api/file`, `/api/close`, `/api/raw`; renderer.js virtualization, tabs.js, history.js, cursor.js, status.js, metrics.go, themes + theme.js, style.css.
4. **Finding things:** fuzzy.go, search.go, symbols.go, `/api/find`, `/api/search`, `/api/outline`, `/api/def`; palette.js, search.js, outline.js, find.js, shortcuts.js with the help sheet and OS-aware labels.
5. **Git:** git.go, status overlay, `/api/diff`, `/api/gutter`; diff.js, gutter decorations, changed-files filter.
6. **Markdown:** markdown.go, `/api/markdown`; markdown.js with the sanitizer and scroll sync.
7. **LSP:** lsp.go, lspservers.go, lspnav.go, calls.go, lspsetup.go, all `/api/lsp/*`; lsp.js, hover.js, inspector.js, calls.js, lspsetup.js.
8. **Agent editing + undo:** settings.go, agent.go, agent_undo.go, `/api/agent/*`; selbar.js, agent.js.
9. **Distribution:** update.go, telemetry.go, Makefile, build.sh, install.sh, release workflow, benchmark.sh.
10. **Hardening and docs:** the full security checklist, end-to-end tests, performance check on a large repo (clone kubernetes; confirm index time around 150 ms and idle RSS around 30 MB; profile with pprof and fix hot spots), README and internals docs. Commit `web/app.js`.

## 11. Definition of done
- `go vet ./...` is clean. `go test ./...` passes. `CGO_ENABLED=0 go build` succeeds for all 15 targets.
- `go build .` works on a machine without Node.
- Running `scry` in a large repository shows the tree instantly. Every feature in Part A works in Chrome, Firefox, and Safari. Nothing is written into the workspace except by the harness or an undo.
- All shortcuts work, with OS-correct labels.
- Documentation matches the code, including the HTTP methods in the API table.

(Execution is driven by the step prompts in .github/prompts/, not by this section.)


## 12. Appendix: Feature inventory (acceptance checklist)


### A1. CLI and process
- [ ] `scry [flags] [file|dir]`. The default target is `.`
- [ ] `scry path/to/file` opens that file. The workspace root is the file's git top-level, otherwise the cwd (if the file is under it), otherwise the file's parent directory
- [ ] `scry file:42` and `scry file:42:7` jump to a line. The suffix is parsed only when the literal path doesn't exist
- [ ] Flags: `-port` (7777; `0` = OS picks), `-host` (127.0.0.1), `-no-open`, `-no-lsp`, `-no-git`, `-dev DIR`, `-agent SPEC`, `-no-agent`, `-no-telemetry`, `-no-color`, `-quiet`, `-update`, `-version`/`-v`, and a `version` subcommand
- [ ] If the port is busy, try the next 99 ports, then fall back to an OS-assigned port
- [ ] Opens the browser without blocking: honors `$BROWSER`; uses `open` on macOS; `rundll32`/`cmd start` on Windows; `xdg-open`/`sensible-browser`/`gio`/common browsers on Linux; `wslview`/PowerShell on WSL after a 500 ms delay
- [ ] Clean terminal banner: workspace, URL, and agent, with color only on a TTY
- [ ] Graceful shutdown on SIGINT/SIGTERM with a 2 s timeout. A second signal exits immediately with code 130. Language servers are shut down and a running agent is cancelled on exit
- [ ] Startup is under 1 ms to listen. Everything else runs in the background

### A2. Workspace index
- [ ] Concurrent directory walk with `NumCPU*4` goroutines, recursing inline when the pool is saturated
- [ ] Full `.gitignore` semantics at every level: anchored/unanchored patterns, `dir/`, `**`, `!negation`, and a literal fast path
- [ ] Ignored entries are listed (dimmed) but never walked or indexed. `.git`, `.hg`, and `.svn` are hidden entirely
- [ ] Symlinks are skipped
- [ ] The root listing is published before the deep walk finishes. The tree endpoint waits up to 300 ms for an unscanned directory
- [ ] Ignored directories can still be browsed: their contents are read from disk on demand and all marked ignored
- [ ] Git status letters overlay file nodes, and every ancestor of a changed file is marked `dirty`
- [ ] Re-index on demand

### A3. Navigation and search
- [ ] Fuzzy file finder: fzf-style two-pass tight matching, boundary/camelCase/basename bonuses, parallel scoring, and match positions for highlighting
- [ ] Workspace search: literal (fast path), regex, whole-word, case-sensitive, and a glob path filter. Runs in parallel with pooled buffers, skips binary files and files over 8 MB, and caps at 200 files × 50 matches with a `truncated` flag
- [ ] Search snippets come pre-split into `pre`/`mid`/`post` with elided lead-in and a 240-rune cap
- [ ] Case-insensitive literal search folds ASCII only, which keeps offsets aligned
- [ ] A glob naming exactly one gitignored file searches that file anyway (so find-in-file works on ignored files)
- [ ] Regex symbol outline for about 13 language families plus Markdown headings
- [ ] Regex go-to-definition fallback: a whole-word search keeping only lines that look like declarations, ranked by filename match, with a reference count

### A4. Viewing
- [ ] Syntax highlighting for about 280 languages through Chroma
- [ ] Windowed lexing: 1,000-line chunks with 400 lines of context on each side, and a 512 KB window cap that falls back to plain text
- [ ] A background full-file pass for files of 2 MB or less. Chunks served before it finishes are marked `refine` and re-fetched by the client
- [ ] Highlight cache: an LRU with a 512 MB budget, charged as chunks render, keyed by `path|mtime|size`
- [ ] Refuses files over 64 MB and binary files. Normalizes CRLF
- [ ] Image preview (png, jpg, gif, webp, svg, ico, bmp, avif)
- [ ] Virtualized DOM: only visible rows are mounted, so a 400k-line file costs the same as a 10-line file
- [ ] Word wrap (`Alt+Z`) and line numbers (`Alt+L`), persisted
- [ ] Caret, click and double-click selection, and occurrence highlighting
- [ ] In-file find (`Cmd/Ctrl+F`) with minimap hit markers
- [ ] Jump to line (`Cmd/Ctrl+G`)
- [ ] Tabs, breadcrumbs, reopen closed tab (`Alt+Shift+T`), back/forward history
- [ ] Markdown preview (GFM + footnotes, GitHub-style heading IDs, Chroma-highlighted fences, raw HTML sanitized in the browser against an allowlist, source-line anchors so switching with `Alt+M` keeps scroll)
- [ ] 14 themes, one CSS file each, concatenated by the server

### A5. Git
- [ ] Detection is memoized and fails quietly. `-no-git` disables it
- [ ] Status from porcelain v2 `-z`: `M A D R C ?` and `!` for conflicts
- [ ] Tree badges, dirty folders, and a changed-files filter
- [ ] Change gutter with added, modified, and deleted markers
- [ ] Diff against HEAD in split or unified layout (`Cmd/Ctrl+D`), remembering the last layout

### A6. Language servers (optional)
- [ ] Ordered registry: gopls, rust-analyzer, pyright > pylsp > ruff, typescript-language-server, clangd, zls, lua-language-server, solargraph, jdtls, omnisharp, texlab
- [ ] Background discovery on `PATH` plus common install directories that are often missing from it
- [ ] Lazy spawn, deduplicated concurrent starts, 30 s handshake, up to 3 crash restarts
- [ ] States: off, starting, indexing, ready, failed. Shown in the status bar
- [ ] Warm on file open
- [ ] Go to definition (`F12`, `Cmd`-click), references (`Shift+F12`), hover cards, document symbols, call trails (`Alt+Shift+H`, callers and callees expanded level by level)
- [ ] UTF-16 ↔ byte column conversion
- [ ] Allowlist of files outside the workspace that a language server pointed at (for example the stdlib), capped at 20k entries
- [ ] Install from the UI: per-OS recipes, auto-run only for user-level installers, sudo recipes displayed but not run, output tail shown, rescan and start afterwards

### A7. Agent editing
- [ ] Harness presets: `claude`, `gemini`, `cursor-agent` (headless, auto-accept edits), plus custom templates containing `{prompt}`
- [ ] Nothing runs until a harness is chosen. The choice is remembered in `~/.scry/settings.json`. `-agent` pins it; `-no-agent` disables editing
- [ ] Select code in the source or diff view (either side, including deleted lines), then use the right-click menu, the footer bar, or `Alt+E` to open the instruction composer
- [ ] The prompt contains the file, line range, fenced snippet, instruction, and "edit in place, change only that, don't explain"
- [ ] One edit at a time, a 10-minute timeout, empty stdin, a 32 KB stdout/stderr tail, and cancel
- [ ] Refuses to edit a file with uncommitted changes unless forced (the UI confirms). The diff view forces automatically
- [ ] Changed files are detected by comparing git-status + size + mtime snapshots before and after the run
- [ ] After a run: evict the highlight cache, close LSP docs, re-index, and reload open tabs in their current mode
- [ ] Errors are shown inline with stdout and stderr. The instruction is preserved
- [ ] Footer shows `Agent: <name>` with a switch menu
- [ ] Undo of the last edit: restore dirty files from pre-run copies (8 MB per file / 64 MB total), restore clean files from the HEAD blob, and delete created paths. All or nothing, used once, and refused if any path changed since the run unless forced

### A8. Selection actions
- [ ] Copy Ref (`Alt+C`), Copy for Agent (`Alt+A`, the same snippet format as the prompt), Find Usages (`Alt+U`), Edit with Agent (`Alt+E`)
- [ ] Footer selection bar and right-click menu

### A9. Palette and shortcuts
- [ ] Universal palette (`Cmd/Ctrl+K`), go to file (`Cmd/Ctrl+P`), command palette (`Cmd/Ctrl+Shift+P`), go to symbol (`Cmd/Ctrl+Shift+O`), workspace search (`Cmd/Ctrl+Shift+F`)
- [ ] Toggle sidebar (`Cmd/Ctrl+B`), close tab (`Alt+W`), `Ctrl+Tab`, `Alt+1..9`, `Alt+Left/Right`, `?` help sheet
- [ ] Shortcut labels adapt to the OS (⌘⌥⇧ glyphs on macOS)

### A10. Operations
- [ ] Gzip on every response with pooled writers; `Cache-Control: no-store`
- [ ] Idle memory scavenger (`FreeOSMemory` after 15 s idle); closing a tab frees memory
- [ ] Path sandbox; `localPost` guard (POST + IP/localhost Host + Origin == Host) on every mutating endpoint
- [ ] Status bar shows RSS, CPU, index time, and file count
- [ ] Self-update (`--update`): SHA-256 checksum required, smoke-test the new binary, atomic replace (Windows `.old` dance), and a daily background check
- [ ] Anonymous opt-out telemetry (only when a key is compiled in; `DO_NOT_TRACK`, `SCRY_TELEMETRY=0`)
- [ ] Single static binary with embedded assets. Cross-compile to 15 targets. GitHub release workflow with checksums. `curl | sh` installer


