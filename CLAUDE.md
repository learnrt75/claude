# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT Project Management Kanban board for an internal IT PMO — an **internal
demo/training tool**, not a real system. The entire app is one file: `index.html`.

There is no git repo, no package manager, no build step, no test suite, and no server.

## Running it

```sh
open index.html          # macOS — or just double-click the file
```

It must keep working from a bare `file://` URL. There is nothing to build, install, or serve;
if you find yourself wanting a dev server or a bundler, that is a signal you are about to break
a hard constraint below.

Verify changes by opening the file and exercising the board by hand. The constraint checks that
matter can be run as greps:

```sh
grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|\balert\(|\bconfirm\(|!important' index.html
grep -oE 'https?://[^"'"'"' )]+' index.html     # must only ever print the formsubmit.co endpoint
grep -nE '<script[^>]+src=|<link[^>]+href=|<img' index.html   # must print nothing
```

(The first grep currently matches two *comments* that name these APIs to explain their absence —
check the hits are comments, not code.)

## Hard constraints

These are product requirements, not style preferences. Breaking one silently defeats the point
of the exercise:

- **Vanilla HTML/CSS/JS only.** No React, Vue, jQuery, Tailwind, npm, bundler, or build step.
- **One file.** All markup, one `<style>` block, one `<script>` block in `index.html`.
- **Zero external resources.** No CDN scripts, no web fonts, no image files. Icons are Unicode
  glyphs or inline SVG; typography is a system font stack. The *only* permitted network call in
  the whole file is the FormSubmit endpoint.
- **No persistence of any kind** — no `localStorage`, `sessionStorage`, IndexedDB, or cookies.
  A refresh resets the board to the seeded demo data; that is intended, and the header says so.
  Do not "helpfully" add persistence.
- **No `alert()` or `confirm()`.** Validation errors render inline under each field; delete
  confirmation is an inline "Delete? Yes / No" row inside the card.
- **No `!important`** in the CSS.
- **Neutral branding.** Text wordmark "UOB IT PMO" and a generic corporate blue palette only.
  No real logos, trademarks, or imitation of any official UOB system.

## Architecture

Both the `<style>` and `<script>` blocks are divided by numbered banner comments
(`1. DESIGN TOKENS`, `7. RENDERING`, …). Grep for those banners to navigate; keep new code
inside the section it belongs to rather than appending to the end.

### State and the render cycle

`state` (script section 3) is the single source of truth:

```js
const state = { tasks: [], filters: {...}, confirmingDelete: null };
```

The whole UI is a pure function of that object. Every mutation follows the same shape: change
`state`, then call `renderBoard()`, which rebuilds all four column bodies from
`renderCard()` and refreshes the count badges and header summary.

**Do not mutate card DOM directly.** This is the rule most likely to be broken by a well-meaning
change. Transient UI states live in `state` and re-render — the inline delete confirmation is
driven by `state.confirmingDelete`, not by toggling a class on a node. `buildColumns()` runs once
at start-up to create the four column shells; after that only column *bodies* are rewritten.

### Event handling

All card interaction is **delegated from the `#board` element** in `wireEvents()` — there are no
per-card listeners, because cards are destroyed and recreated on every render. Buttons and the
move `<select>` carry `data-action` + `data-id`, and the delegated handler switches on
`data-action`. New card controls must follow this pattern.

Every drag-and-drop interaction has a keyboard-accessible twin: the native HTML5 DnD handlers and
the per-card "Move ▸" `<select>` both funnel into `moveTask()`. Keep that pairing — the board is
required to be fully operable without a mouse.

### Escaping

`renderCard()` and friends build HTML strings and assign them via `innerHTML`, so **every
interpolated value must go through `escapeHtml()`** — including values that look like they come
from a fixed list. Toast text uses `textContent` instead and needs no escaping.

### Derived values

Nothing about counts, overdue status, or filtering is stored; it is all recomputed from
`state.tasks` on each render via `isOverdue()`, `applyFilters()`, and `renderSummary()`. Dates are
local-time `YYYY-MM-DD` strings compared lexically (`toISODate()`, `todayISO()`) — do not switch
to `Date` object comparison or `toISOString()`, which reintroduces UTC drift.

Seed data (section 5) expresses due dates as **day offsets from today**, not literals, so that
overdue cards are always demonstrable whatever day the file is opened.

### Shared option lists

`STATUSES`, `PROJECTS`, `CATEGORIES`, and `PRIORITIES` (section 2) drive the form selects, the
filter selects, the four columns, and validation. Add a new project or category in that one array
only — the selects and `validateForm()` both read from it, so they cannot drift apart.

## FormSubmit integration

`notifyNewTask()` is the only network call. Two things about it are deliberate:

- **`FORMSUBMIT_ENDPOINT`** (script section 1) is the single place the target email address
  appears, so it can be swapped in one edit. It ships with the literal `YOUR_EMAIL@example.com`
  placeholder. FormSubmit requires a **one-time activation**: the first submission triggers a
  confirmation email to that address, and nothing is delivered until the link in it is clicked.
- **The board is optimistic and the call cannot break it.** `handleSubmit()` adds the card,
  re-renders, and resets the form *before* awaiting the fetch; the fetch is wrapped in
  `try/catch` and a failure only produces a warning toast. Never make card creation depend on the
  request succeeding.

Never send the user's email address anywhere except this endpoint.

## Publishing

The board has also been published as a Claude Artifact at
`https://claude.ai/code/artifact/6f44fa78-de7d-4224-80a2-1aeafb1e4e80`.

That copy is generated from `index.html` by stripping the outer `<!doctype>`, `<html>`, `<head>`,
and `<body>` tags (the artifact platform supplies its own skeleton) and shortening the `<title>`.
Republish by rebuilding that stripped copy and passing the URL above, so the link is preserved.
Note that the artifact sandbox blocks the FormSubmit fetch, so the email path can only be
exercised from the local file — in the hosted copy it always falls through to the
"Card added locally" warning toast, which is correct behaviour.
