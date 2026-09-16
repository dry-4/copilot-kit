---
mode: agent
description: "scry step 12: self-update, telemetry, Makefile, cross-builds, installer, release CI, benchmarks"
---
# Step 12: Distribution

Read `HANDOFF.md` first.

## Read from SPEC.md
§4.14 (`update.go`, `telemetry.go`), §6 (everything except `build-web.js`, which already exists).

## Build
1. `update.go`: `githubRelease`, `updateState`, `getRepoName` (`SCRY_REPO` or `scry-app/scry`), `stateFilePath`, `read`/`writeUpdateState`, `compareSemver`, `fetchLatestRelease` (with a timeout and a `SCRY_UPDATE_API` base URL override for tests), `downloadAsset`, `checksumFor(data, assetName)` (sha256sum format), `downloadVerifiedAsset`, `checkDailyUpdate`, `printUpdateNotification`, `runSelfUpdate` (per §4.14), `copyOrMove`. Wire `-update` and the background daily check in `main.go`.
2. `telemetry.go` per §4.14. Wire it into `main.go`: `session_started` after indexing; `Close("normal")` or `Close("interrupted")` on exit. Add `var posthogKey string` for ldflags injection.
3. `Makefile`: targets `all`, `help`, `web`, `build`, `test`, `dist`, `publish <version>`, `clean`. `POSTHOG_KEY` is optionally injected via ldflags.
4. `build.sh`: the 15 targets, `CGO_ENABLED=0`, `-trimpath -ldflags "-s -w"`, names `scry-<ver>-<os>-<arch>[.exe]`.
5. `install.sh` (POSIX sh): OS and arch detection, `VERSION` (latest via the API), download of the binary and `checksums.txt`, verification with `sha256sum` or `shasum -a 256`, install to `INSTALL_DIR` (`/usr/local/bin` if writable, else `~/.local/bin` with a PATH hint), `SCRY_REPO`, colored log helpers.
6. `.github/workflows/release.yml` per §6. **Use the same Go version as `go.mod`** (read it with `go-version-file: go.mod`).
7. `benchmark.sh` with modes `--clone`, the default, `--vscode DIR`, `--memory DIR`, `--lsp DIR` per §6. It must use `scry -no-open -port 0 -quiet -no-telemetry` and discover the port from stdout.
8. Make sure every target compiles: the metrics and tty build tags must cover windows, freebsd, openbsd, netbsd, linux/riscv64, and linux/arm.

## Tests
- `compareSemver`: `0.1.10 > 0.1.9`, a leading `v`, an equal pair, a prerelease suffix is handled sensibly.
- `checksumFor`: finds the right line; a missing asset gives an error.
- `downloadVerifiedAsset` against an `httptest` server: a good checksum passes; a mismatch returns an error and writes nothing usable; a missing `checksums.txt` makes `runSelfUpdate` fail (use the API override).
- Telemetry: disabled without a key; each opt-out env var (`DO_NOT_TRACK=1`, `SCRY_TELEMETRY=off`) disables it; the `filesBucket` boundaries; with a key and an `httptest` host, `Track` then `Close` delivers `session_started` and `session_ended` with no paths in the payload.

## Verify
```
go test ./... 2>&1 | tail -20
make dist && ls dist | wc -l        # expect 15
sh -n install.sh && sh -n build.sh && bash -n benchmark.sh
./dist/scry-*-$(go env GOOS)-$(go env GOARCH) -version
```
Then follow the definition of done in copilot-instructions.md.
