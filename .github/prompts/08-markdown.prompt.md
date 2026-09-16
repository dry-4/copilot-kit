---
mode: agent
description: "scry step 08: Markdown preview (goldmark backend + sanitizing frontend)"
---
# Step 08: Markdown preview

Read `HANDOFF.md` first.

## Read from SPEC.md
§4.8, §4.13 (row `/api/markdown` and the `markdown` field of `/api/file`), §5.2 (`markdown.js`).

## Build
1. `go get github.com/yuin/goldmark`.
2. `markdown.go`:
   - `isMarkdown`; the converter with GFM, Footnote, auto heading IDs, `html.WithUnsafe()`
   - `headingIDs`, a GitHub-compatible ID generator (lowercase; keep Unicode letters, digits, `-`, `_`; spaces become `-`; drop everything else; `-1`, `-2` suffixes for duplicates)
   - a `lineMarker` AST transformer setting `data-line` on block nodes
   - a fenced-code renderer highlighting with Chroma using the same class names as `highlight.go` (reuse `classFor`), plain above 256 KB
   - a 4 MB cap; `/api/markdown`; `markdown: isMarkdown(rel)` in `/api/file`
3. `markdown.js`:
   - `#mdview` / `article#md`, shown when the tab is Markdown and preview is on (the default; persisted in `scry.mdPreview`).
   - **Sanitizer:** parse the HTML with `<template>` and rebuild a new DOM copying only allowlisted elements (headings, p, a, img, lists, table parts, code, pre, blockquote, em, strong, del, hr, br, details, summary, sup, sub, div, span, input[type=checkbox][disabled]) and attributes (href, src, alt, title, id, class, align, width, height, colspan, rowspan, data-line, checked, disabled, open). Drop `on*` attributes, `style`, `javascript:`/`data:` URLs (allow `data:image/*` for img), and anything else.
   - Links: `#anchor` scrolls inside the preview; relative `.md` paths inside the workspace open in scry (at the anchor if present); other relative paths open the file; external links get `target=_blank rel=noopener`.
   - Relative image paths are resolved against the Markdown file's directory and rewritten to `/api/raw?path=<resolved>`.
   - `Alt+M` (`hooks.toggleMarkdown`) and `#md-switch` toggle Preview/Source. When switching, keep the position: source top line maps to the nearest `data-line` element, and back again. Export `syncPreview`, `togglePreview`, `previewing`, `previewLine`, `previewTopLine`.
   - The status bar shows the Preview button only on Markdown tabs (`body.md-tab`).
4. `.md` preview styles in `style.css`, using theme variables.

## Tests
- Heading IDs: `Hello World` → `hello-world`; duplicates → `-1`, `-2`; `Überblick` → `überblick`; `API_Reference!` → `api_reference`.
- Blocks carry `data-line` with correct 1-based lines.
- A fenced `go` block contains `<i class="k">` for `func`.
- Files over the cap are rejected. `/api/markdown` returns `{path, html}`.

## Verify (manual; report)
Open a README with tables, task lists, a centered HTML logo, code fences, relative links and images: rendering, links, images, and Preview/Source scroll sync all work. Check that `<img src=x onerror=alert(1)>` and `<script>` in a test .md file do nothing.

Then follow the definition of done in copilot-instructions.md.
