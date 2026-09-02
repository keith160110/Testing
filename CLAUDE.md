# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

LogiTrack Inventory — a warehouse stock dashboard for an operations coordinator, living entirely
in [index.html](index.html). Markup, CSS and JavaScript are all inline in that one file; there is
no second source file and no `package.json`.

## Hard constraints

These are requirements of the deliverable, not incidental choices. Do not relax them without the
user saying so explicitly:

- **One file.** Everything ships in `index.html`. No extracted `.css`/`.js`, no imports.
- **No build step, no server.** It must work by double-clicking the file (`file://` origin).
- **No frameworks, no external libraries, no CDN links, no fonts or assets fetched over the
  network.** The page must make zero network requests.
- **No `localStorage`/`sessionStorage`/IndexedDB or any storage API.** State is in-memory only and
  is expected to reset on reload.
- **No `alert()`/`confirm()`/`prompt()`.** All feedback goes through the `#liveMessage` aria-live
  region (`showMessage()`) or the per-field `.error` elements.
- **No inline `onclick`/`on*` attributes.** Everything is wired in `bindEvents()`.

## Architecture

The script is divided by banner comments into six sections; keep new code inside the matching one.

1. **State** — `inventory` (the array of records) is the single source of truth, plus `uiState`
   (search text, category filter, sort key/direction), the `CATEGORIES`/`WAREHOUSES` option lists,
   and the `el` object of cached DOM references.
2. **Derivation + formatting helpers** — `getStatus()`, the `Intl` formatters, `escapeHtml()`,
   `matchesFilters()`, `getVisibleRecords()`.
3. **Render functions** — `renderTable()`, `renderSortIndicators()`, `renderSummary()`,
   `renderAll()`, `showMessage()`, `flashRow()`.
4. **Validation** — `setFieldError()`, `clearAllErrors()`, `readNumber()`, `validateForm()`.
5. **Event handlers** — one function per interaction, no logic inline in `addEventListener`.
6. **Initialisation** — `populateSelects()`, `bindEvents()`, `init()`.

The data flow is strictly one-directional: **mutate `inventory` → call `renderAll()` → the DOM is
rebuilt from scratch.** Rules that follow from this and are easy to break:

- Never read application data back out of the DOM. `data-id` on a row/button is a *lookup key* into
  `inventory`, not storage.
- `renderTable()` replaces `tbody.innerHTML` wholesale, so row buttons are destroyed on every
  render — Delete works only because the `click` listener is delegated to `<tbody>`. Any new
  per-row control must be delegated the same way.
- Any string that originated from user input (`sku`, `name`, `supplier`) must pass through
  `escapeHtml()` before being concatenated into the row HTML.
- **Status is derived, never stored.** It is not a field on a record; `getStatus()` computes
  `"out"` / `"low"` / `"in"` from `qty` vs `reorder` on every render. Adding a `status` property to
  a record would create a second source of truth.
- Sorting by Status goes through `STATUS_RANK` (out → low → in), not the display label.
- `renderSummary()` deliberately counts the **whole** `inventory` array, not the filtered view —
  the summary strip is the true warehouse picture regardless of active filters.
- Filter/sort changes call `renderTable()` alone; anything that changes `inventory` must call
  `renderAll()` so the summary follows.

Records are `{ id, sku, name, category, warehouse, qty, reorder, cost, supplier, updated }`.
`id` is an internal integer from the `nextId` counter and is never displayed; `sku` is the
user-facing identifier and is enforced unique case-insensitively in `validateForm()`.
`updated` is a `YYYY-MM-DD` string, which is why plain string comparison sorts that column
correctly.

The `CATEGORIES` and `WAREHOUSES` constants feed both the table's filter dropdown and the form's
selects via `populateSelects()` — add an option there, not in the HTML.

## Visual design

The page carries **Kacific** branding. The two brand colours are sampled straight from the logo —
`--kac-blue: #034ea2` (wordmark) and `--kac-grey: #a7a9ac` (swoosh) — and every other blue in the
palette derives from them. Do not introduce a blue that is not a token.

- **The logo is an inline `data:image/png;base64` URI** on `.brand-logo`. That is deliberate: it is
  the only way to show the mark while the page still makes zero network requests. The source base64
  lives in `logo-kacific.b64` at the repo root — regenerate the `src` from that file rather than
  pasting a new blob by hand. The mark is drawn in brand blue, so it always needs the white chip
  behind it and must never sit directly on the blue header.
- **The body font is a Comic face** (`--font`), resolved from locally installed fonts only
  (`Comic Sans MS` → `Comic Neue` → `Chalkboard SE` → `cursive`). No webfont, by constraint. Comic
  faces run wide and loose, which is why headings carry negative letter-spacing and every numeric
  cell keeps `font-variant-numeric: tabular-nums` — drop those and the table columns stop aligning.
- **The filter bar (`.toolbar`) is light blue.** Its search and select carry that tint through
  (`--filter-field`) instead of sitting on it as white boxes. The `--filter-*` tokens exist only for
  this strip and the table head; they are not general-purpose.
- **Layout order is form-left, records-right.** The form `<section>` comes first in the DOM so
  keyboard focus order matches the visual order both on desktop and once the grid stacks below
  900px. If you swap the columns visually, move the DOM nodes — do not reach for CSS `order`.
- `.card-form` is `position: sticky` on desktop and reset to `static` in the 900px query.
- CSS section 12 is `prefers-reduced-motion`; there, the new-row flash degrades to a static tint.

The header's cropped ellipses (`.topbar::before` / `::after`) echo the swoosh in the logo. They are
the one decorative flourish on the page — everything else stays plain so the data reads first.

## Running and checking changes

There is no build, lint, or test tooling, and no test framework is set up.

```powershell
Invoke-Item index.html          # open in the default browser
```

Verify by hand in the browser (DevTools console should stay clean, Network tab empty):
sort each column both directions; search + category filter together and with no matches
("No records match"); submit the empty form (eight inline errors, no dialog); submit a duplicate
SKU; add a record with `qty` 0 and one where `qty` equals `reorder` (Out of Stock / Low Stock);
delete a row and confirm the summary strip and the confirmation message both update; resize below
900px to confirm the layout stacks and the table scrolls inside its card rather than widening the
page.

Note that no JavaScript runtime is installed on this machine (`node` is not on PATH), so the inline
script cannot be syntax-checked outside a browser.

## Published artifact

The app is also published as a Claude Artifact. Its URL is kept out of this file because this
repository is public — read it from `.claude/artifact-url.txt`, which is git-ignored and stays on
the local machine.

To update it after editing `index.html`: the Artifact publisher supplies its own
`<!doctype>/<html>/<head>/<body>` wrapper, so the published copy is `index.html` with those wrapper
lines (and the two `<meta>` tags) stripped, keeping `<title>` and `<style>` at the top. Regenerate
that variant into the scratchpad, then publish it with that saved URL as `url` so it updates in
place instead of creating a second artifact.

## Other agent configs

A user-level OpenAI Codex config exists at `~/.codex/config.toml`. If you want its importable
pieces (MCP servers, slash commands, subagents, skills, instructions) available in Claude Code,
reply `/import` to see a scan of what's importable, then `/import --yes=<digest>` using the digest
that scan prints. If `/import` isn't available on this surface, run `claude import` from a terminal.
