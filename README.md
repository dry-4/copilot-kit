# scry Build Kit for GitHub Copilot (Agent mode, Claude Opus 5)

A ready-to-copy kit for rebuilding scry from scratch with GitHub Copilot in VS Code. The work is split into **17 small steps**, each run in its own chat. Each step is sized to fit comfortably in one context window and ends tested and committed.

## What's in the kit

```
copilot-kit/
├── SPEC.md                              Full specification (§1–§11) + feature checklist (§12)
├── .github/
│   ├── copilot-instructions.md          Auto-loaded into every Copilot chat: context rules,
│   │                                    engineering rules, definition of done, HANDOFF format
│   └── prompts/
│       ├── 01-skeleton.prompt.md        ┐
│       ├── 02-index-tree.prompt.md      │ Part 1: foundation and viewer
│       ├── 03-highlight-backend.prompt.md
│       ├── 04-viewer-ui.prompt.md       ┘
│       ├── 05-search-backend.prompt.md  ┐
│       ├── 06-navigation-ui.prompt.md   │ Part 2: navigation, git, markdown
│       ├── 07-git.prompt.md             │
│       ├── 08-markdown.prompt.md        ┘
│       ├── 09-lsp-core.prompt.md        ┐
│       ├── 10-lsp-features.prompt.md    │ Part 3: language servers, distribution
│       ├── 11-lsp-ui.prompt.md          │
│       ├── 12-distribution.prompt.md    ┘
│       ├── 13-agent-dispatch.prompt.md  ┐
│       ├── 14-undo.prompt.md            │ Part 4: agent editing, hardening, docs
│       ├── 15-agent-ui.prompt.md        │
│       ├── 16-hardening-e2e.prompt.md   │
│       ├── 17-docs-final.prompt.md      ┘
│       └── resume.prompt.md             Recover an interrupted or broken step
└── README.md                            This file (not needed in the new repo)
```

State passes between chats only through **`HANDOFF.md`**, which each step rewrites, and **git commits**. No chat depends on an earlier conversation.

---

## 1. One-time setup on the other machine

### Tools
- VS Code with the **GitHub Copilot** and **GitHub Copilot Chat** extensions, signed in, on a plan that offers Claude Opus 5
- Go (current stable), git, and Node.js 20+ (or Bun), which are only used to bundle the frontend
- Optional, for step 11: `gopls` (`go install golang.org/x/tools/gopls@latest`)

### Create the repo
```bash
mkdir scry && cd scry && git init
# copy the kit contents (not the README) in:
cp /path/to/copilot-kit/SPEC.md .
cp -r /path/to/copilot-kit/.github .
git add -A && git commit -m "Add spec and Copilot build kit"
code .
```

### VS Code settings
Open your workspace settings (`.vscode/settings.json`) and set these. Setting names change between VS Code releases; if one isn't recognized, search the Settings UI for the words in the comment.

```jsonc
{
  // Load .github/copilot-instructions.md automatically ("instruction files")
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  // Enable .github/prompts/*.prompt.md ("prompt files")
  "chat.promptFiles": true,
  // Let agent mode run longer before asking "Continue?" ("agent max requests")
  "chat.agent.maxRequests": 100,
  // Auto-approve routine terminal commands so the agent doesn't stall on every test run
  // ("terminal auto approve"). Keep destructive commands like rm or git push out of this list.
  "chat.tools.terminal.autoApprove": {
    "go": true, "gofmt": true, "node": true, "bun": true,
    "git status": true, "git add": true, "git commit": true, "git log": true, "git diff": true,
    "git init": true, "curl": true, "make": true, "grep": true, "ls": true, "sh -n": true
  }
}
```

If your VS Code version reports that `mode: agent` in the prompt-file header is deprecated, replace it with `agent: agent` in all 18 prompt files: `sed -i 's/^mode: agent$/agent: agent/' .github/prompts/*.prompt.md`.

---

## 2. How to run each step

For every step, in order `01` → `17`:

1. **Open a new chat.** Use the `+` button, not an existing chat. This matters more than anything else in the kit.
2. Set the chat mode dropdown to **Agent** and the model picker to **Claude Opus 5**.
3. Type `/` and pick the step, for example `/01-skeleton`, then press Enter. If slash commands don't list the prompt files, open the `.prompt.md` file and click the ▶ run button in its editor title bar, or paste its body into the chat.
4. Let it work. Approve anything not auto-approved. If it asks **"Continue?"**, click Continue.
5. When it finishes, **check before moving on**:
   - `git log --oneline | head -3` shows a `Step NN: …` commit
   - `go test ./... 2>&1 | tail -5` passes
   - `HANDOFF.md` looks sensible (skim it)
   - for UI steps (04, 06, 07, 08, 11, 15): run `go run . .`, open the browser, and try the step's "Verify" list yourself
6. Only then start the next step, in a **new chat**.

### If a chat stops early, errors out, or gets confused
- Open a new chat and run `/resume`. It works out the current step from git and `HANDOFF.md` and finishes it.
- You can add a note after the command, for example: `/resume the diff view selection mapping is off by one on deleted lines`.
- If the chat's answers start contradicting earlier work (a sign its context has been summarized), stop it and use `/resume` in a fresh chat.

---

## 3. Step map

| # | Step | Kind | Risk | You should see afterwards |
| --- | --- | --- | --- | --- |
| 01 | Skeleton, CLI, server core | Go | Low | `go run . -no-open .` prints a URL; `/api/meta` JSON |
| 02 | gitignore, index, file tree | Go + UI | Medium | Tree in the browser, ignored entries dimmed |
| 03 | Highlighting backend | Go | Medium | `/api/file` returns highlighted HTML lines |
| 04 | Virtualized viewer, tabs, themes | UI | **High** | Smooth scrolling through a 200k-line file; 14 themes |
| 05 | Fuzzy find, search, outline | Go | Low | `/api/find` and `/api/search` fast and correct |
| 06 | Palette, search panel, find, shortcuts | UI | Medium | Cmd/Ctrl+K/P/Shift+F/F work; `?` help sheet |
| 07 | Git status, gutter, diff view | Go + UI | Medium | Badges, gutter, split and unified diff |
| 08 | Markdown preview | Go + UI | Medium | README renders safely; Alt+M keeps position |
| 09 | LSP client core | Go | **High** | Fake-server tests pass; gopls starts |
| 10 | LSP features, install flow | Go | Medium | Definition into stdlib works via curl |
| 11 | LSP UI | UI | Medium | F12, hover, references, call trail |
| 12 | Update, telemetry, packaging | Go + scripts | Low | `make dist` builds 15 binaries |
| 13 | Agent dispatch | Go | Medium | Fake-harness tests pass |
| 14 | Undo | Go | Medium | All undo tests pass |
| 15 | Agent UI | UI | **High** | Edit and undo from the source and diff views |
| 16 | Hardening, E2E, performance | Go | Medium | Security table + performance numbers |
| 17 | Docs + acceptance | Docs | Low | README, internals docs, `ACCEPTANCE.md` |

---

## 4. Getting the best results from Copilot

- **One step per chat, always.** Copilot summarizes long conversations to fit its window, and summaries lose details like function names and JSON field names. Short chats avoid that.
- **Don't skip the checks between steps.** A wrong decision in step 04 (how virtual rows work) or step 07 (diff row attributes) costs little to fix right away and a lot to fix in step 15.
- **Keep `SPEC.md` the source of truth.** To change a feature, edit `SPEC.md` first and commit. Don't rely on telling a chat, because the next chat won't see it.
- **Steer with short follow-ups inside the same chat:** "run the failing test with -run and fix it", "you're reading too much; use grep and line ranges", "stop and update HANDOFF.md now".
- **Watch for scope creep.** If a chat starts building a later step's feature, tell it to stop and leave a stub.
- **Keep an eye on premium requests.** Opus 5 is a premium model in Copilot, and 17 agent sessions use a meaningful number of requests. Steps 01, 05, 12 and 17 are the most mechanical, so a cheaper model is a reasonable choice there if you need to save.
- **Don't use Copilot's cloud coding agent** (assigning GitHub issues) for UI steps, because it can't check the browser UI. It's usable for backend-only steps (03, 05, 09, 10, 12, 13, 14, 16) if you set up Go and Node in its environment, but VS Code Agent mode is the smoother path for this kit.

## 5. When it's done

`ACCEPTANCE.md` (written in step 17) lists every feature with evidence and a ✅/⚠️/❌ status. Work through any ⚠️ or ❌ items with `/resume <describe the item>` in fresh chats.
