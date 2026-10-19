# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo/training IT PMO Kanban board for a **fictitious** bank. The whole app is `index.html`: markup, one `<style>` block and one `<script>` block.

## Hard constraints (from the original spec — keep them)

- Vanilla HTML/CSS/JS only. No frameworks, build step, bundler, npm, CDN scripts, web fonts or image files. Use the system font stack and inline SVG/Unicode for icons.
- Must still work when opened by double-clicking (`file://`). No server needed.
- **No persistence.** Don't use `localStorage`, `sessionStorage`, IndexedDB or cookies. A refresh resets to the seed data on purpose, and the header note says so.
- The only network call is FormSubmit's AJAX endpoint (`FORMSUBMIT_ENDPOINT`). No other backend, and no data sent anywhere else.
- Neutral branding only: "IT PMO" wordmark, `ITPM-####` IDs, `[IT PMO]` email subject. Never use real bank names, logos or trademarks.
- No `alert()`/`confirm()`. Validation errors appear inline, and delete uses an inline "Delete? Yes / No" toggle. No `!important`. Colours and spacing come from `:root` custom properties.
- Escape every user-supplied string with `escapeHtml()` before it goes into HTML.

## Running and checking

- Run: open `index.html` in a browser. To test real FormSubmit delivery, serve it with `python3 -m http.server` because FormSubmit may reject `file://` origins. A new address also needs FormSubmit's one-time activation email to be clicked.
- There are no tests or linter. Quick checks:
  - Syntax-check the script: `sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' > /tmp/app.js && node --check /tmp/app.js`
  - Check for forbidden APIs: `grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|https?://|alert\(|confirm\(|!important' index.html` (the FormSubmit URL should be the only hit)
  - Headless render: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --dump-dom file://$PWD/index.html`

## Script architecture

- **Config at the top:** `FORMSUBMIT_ENDPOINT`, `TASK_ID_PREFIX` and `NOTIFY_TIMEOUT_MS`. The email address is changed only here.
- **Reference data:** `STATUSES` (with column descriptions), `PROJECTS`, `CATEGORIES` and `PRIORITIES`. Columns, form selects and filter selects are all generated from these arrays, so add values there.
- **Single source of truth:** `state = { tasks, filters }`. Separately, `uiState` holds per-card UI flags: `confirmDeleteId`, `moveOpenId`, and `focusAfterRender`, a selector to focus after re-render. Task IDs come from the `idCounter` in `nextTaskId()`.
- **One-way flow:** all task changes go through `addTask()`, `moveTask()` or `deleteTask()`, and each calls `renderBoard()`. `renderBoard()` rebuilds the summary, column badges and card lists from state, then calls `restoreFocus()`. Don't change card DOM outside `renderBoard()`/`renderCard()`. Change `uiState` and re-render instead.
- **Dashboard panels:** `renderBoard()` drives everything: `renderKpis`, `renderStatusMix`, `renderProjectChart`, `renderOutlook`, `renderAttention` and `renderColumns`. Charts are plain HTML/CSS marks with no SVG or chart library. Each chart has a Chart/Table toggle (`uiState.views`), and one shared tooltip reads `data-tip`.
- **Status colours (fixed by the brief):** red = Overdue, green = Done, blue = Backlog, orange = In Progress, brown = Blocked, set as `--st-*` tokens with dark-mode variants. `SERIES` order (Done → Backlog → Blocked → In Progress → Overdue) was validated for colour blindness, so don't reorder it. Every status also has an icon (`swatch()`), so colour is never the only cue. Priority pills are neutral on purpose.
- **Filtering:** `applyFilters()` scopes the whole dashboard: KPIs, charts, the attention table, column badges and cards.
- **Events:** card buttons use `data-action` and `data-id`, and one delegated click handler (`handleBoardClick`) dispatches them. Drag and drop uses the native HTML5 API. Listeners go on the board and on the column elements, which `buildColumns()` creates once.
- **Add Task flow:** `handleSubmit` → `validateForm` (inline errors via `setFieldError`) → optimistic `addTask` → `notifyNewTask` in a try/catch. A failed notification only shows a warning toast and never removes the card. While the request is in flight, the submit button is disabled and reads "Sending…" (`setSending`). `notifyNewTask` also treats a 200 response with `success: "false"` as a failure.
- **Dates:** use the local-date helpers (`todayISO`, `addDaysISO`, compared as `YYYY-MM-DD` strings), not UTC `toISOString()`. Seed due dates are relative to today, so the demo always has overdue and upcoming tasks.
