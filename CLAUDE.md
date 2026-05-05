# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, zero-build static web app that lays out travel activities as printable A6 cards (4 per A4 page). Two decks ship: Singapore (`sg`) and Kuala Lumpur (`kl`). Deployed to GitHub Pages from `main` via `.github/workflows/static.yml` — every push to `main` deploys the entire repo as-is.

## Running / "building"

There is no build system, package manager, lint, or test suite. Open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`). JSZip loads from a CDN at runtime, so internet is required for the Export feature.

To regenerate a clean PDF, use the toolbar's "Print / Save as PDF" — print CSS in `index.html` is tuned to 190mm × 273mm content area (the `@page` 10mm margin already accounts for the rest; the few-mm height trim absorbs browser rounding).

## Architecture

### Source of truth at runtime: the embedded `DECKS` object

`index.html` is self-contained. The `DECKS` JS literal near the top of the `<script>` block holds **both** decks inline; `activities-{sg,kl}.json` are mirrors used as inputs for regenerating that literal, not loaded at runtime. This means:

- Editing activity content requires editing the `DECKS` literal in `index.html` (and ideally the matching JSON for consistency).
- Adding a new deck requires: (1) a new key in `DECKS`, (2) a new `<option>` in the `#deck-select`, (3) optionally a sibling `activities-{key}.json`, (4) a `/photos-{key}` entry in `.gitignore` if photos will be committed-out rather than embedded.

### `template.html` ↔ `index.html` relationship

`template.html` is identical to `index.html` except `const DECKS = ...` is replaced with `const DECKS = DECKS_PLACEHOLDER;`. It exists so the embedded `DECKS` can be regenerated from the JSON files without manually pasting. There is no script that performs this substitution in the repo — it's done ad hoc (e.g. `sed`/editor) when refreshing `index.html` from the JSONs. **Keep CSS/JS edits in sync between the two files**, since drift between them silently breaks any future regen.

### Photo storage

Photos are kept in IndexedDB (`activity-cards` DB, `photos` store) keyed as `"{deck}::{activityName}"` — so the same activity name across decks doesn't collide, and switching decks reloads the relevant subset. Photos are never uploaded; the in-browser blobs are turned into object URLs for display and for printing.

`act.photo_url` (a relative path like `./photos-sg/gardens-by-the-bay.jpg`) is the fallback when no IndexedDB blob exists, used after the user has run Export → unzipped next to the HTML. The `/photos-sg` and `/photos-kl` directories are gitignored on purpose: photos are user-supplied and may be copyrighted.

### Export flow (`exportZip`)

Produces a zip containing: `photos-{deck}/<slug>.<ext>` files, an updated `activities-{deck}.json` with `photo_url` rewritten to those paths, and a patched `activity-cards.html` whose embedded `DECKS[currentDeck].activities` has the same rewritten URLs. The patch uses a regex (`/const DECKS = \{[\s\S]*?\};(?=\s*\/\/)/`) that **depends on the comment immediately following the `DECKS` literal** — preserve the `// ...` comment after `const DECKS = {...};` or the export's HTML-patch step will silently no-op.

### Categories drive color

Five categories — `sights`, `food`, `kids`, `nature`, `culture` — map to CSS custom properties (`--c-*`) and class selectors (`.card.sights` etc). Adding a category means adding both a `--c-<name>` variable and a `.card.<name> { --accent: ... }` rule.

## Conventions worth preserving

- A6 card geometry assumes exactly `CARDS_PER_PAGE = 4` in a 2×2 grid; changing this requires updating both the JS constant and the `.page` grid template.
- Activity `name` is the de facto primary key (used for IndexedDB scoping, photo filenames via `slugify`, and the photo-search URL). Renaming an activity orphans its stored photo.
- The print stylesheet relies on `print-color-adjust: exact` so the dark text bands and accent colors survive. Don't drop those declarations when refactoring CSS.
