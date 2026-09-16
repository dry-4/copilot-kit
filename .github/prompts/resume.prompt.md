---
mode: agent
description: "scry: resume an interrupted step or fix a broken state"
---
# Resume or repair

A previous chat stopped before finishing its step, or the build is broken. Do this:

1. Read `HANDOFF.md`. Run `git log --oneline | head -10` and `git status --short | head -30`.
2. Run `go build ./... 2>&1 | tail -20`, `go vet ./... 2>&1 | tail -20`, and `go test ./... 2>&1 | tail -40`.
3. Identify the **current step**: the lowest-numbered step in `.github/prompts/` without a "Step NN" commit. Read that step's prompt file (only that one).
4. Compare what the step requires against what exists (grep for the functions, handlers, and exports it names). Make a short checklist of what is done and what remains.
5. If uncommitted work exists and is coherent, keep it. If it is half-broken, fix it forward. Only `git stash` it when fixing forward is clearly worse, and say so.
6. Finish the remaining items of that step only, then follow the definition of done in `copilot-instructions.md` and commit with the step's title.

If the user added a note below, it takes priority:
