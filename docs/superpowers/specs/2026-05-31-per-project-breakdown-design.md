# Per-project breakdown (drill-down view)

**Status:** Design — not yet implemented
**Date:** 2026-05-31
**Surface:** Dashboard only (no CLI changes)

## Goal

Let the user drill into a single project from the dashboard and see the entire dashboard scoped to that project's activity. Clicking any project name anywhere on the page enters this mode; an explicit "All projects" control exits it.

## User flow

1. From the default ("all projects") view, the user clicks a project name in:
   - the **Top Projects by Tokens** chart bar/label, or
   - the **Cost by Project** table row, or
   - the **Cost by Project & Branch** table row, or
   - the **project** column of any session row in the sessions table.
2. The page enters **project mode**:
   - A header banner appears at the top of the content area: `Viewing project: <name>   [× All projects]`.
   - URL hash updates to `#project=<urlencoded-name>`.
   - The three project-aggregation sections (Top Projects chart, Cost by Project table, Cost by Project & Branch table) are hidden — they're meaningless when scoped to one project.
   - All remaining sections (Totals, daily chart, hourly chart, By Model, sessions table) recompute filtered to just that project's data.
   - The Cost by Project & Branch table is **replaced** by a "Cost by Branch" table showing branches *within* this project (same columns minus "Project").
3. Existing filters (date range, model multi-select) continue to apply on top of the project filter.
4. The user clicks **× All projects** (or removes the hash from the URL) to return to the unscoped view.

Refresh, bookmark, and browser-back all work because the project is encoded in the URL hash.

## Architecture

The dashboard's existing client-side filter pipeline (`applyFilter()`) already filters `daily_by_model`, `hourly_by_model`, and `sessions_all` by date range and model. Adding a project dimension is a third filter clause. The only structural change is that the daily/hourly chart sources don't currently carry a `project_name` column, so the server-side aggregates must include it.

### Backend changes ([dashboard.py](dashboard.py) `get_dashboard_data`)

Two SQL queries grow a JOIN and one extra grouping column:

- `daily_rows`: JOIN `turns t LEFT JOIN sessions s ON t.session_id = s.session_id`, add `COALESCE(s.project_name, 'unknown') as project`, GROUP BY `(day, model, project)`. Emitted list renamed `daily_by_model_project`. No compatibility shim — the `/api/data` payload is internal to this app and the frontend ships in lockstep.
- `hourly_rows`: same JOIN + group-by addition, emit `hourly_by_model_project`.

Cardinality impact: rows grow by ≤ N_projects. In practice a typical user has < 30 projects and ~365 days; worst case ~100K rows in the daily list — well within the existing single-payload model.

`sessions_all` already carries `project`, no change there.

### Frontend changes ([dashboard.py](dashboard.py) `HTML_TEMPLATE`)

**New JS state:**

```
let selectedProject = null;  // string or null
```

**URL persistence** (`updateURL` / new `readURLProject`):
- On load, parse `location.hash` for `project=<encoded>`; set `selectedProject`.
- When entering/leaving project mode, write `history.pushState(...)` with the new hash (pushState, not replaceState, so back-button works). On `popstate`, re-read the hash and re-render.

**Filtering** (`applyFilter`):
- Add `selectedProject` predicate to the three filter sites: filtered daily rows, filtered hourly rows, filtered sessions.
- When `selectedProject` is set, build `byBranch` (project+branch table, but project column dropped) from filtered sessions for the in-project Cost by Branch table.

**Rendering** (top of `applyFilter` or in a small `applyProjectMode()` helper):
- Toggle a body class `project-mode` based on `selectedProject != null`.
- CSS hides `.section--all-projects-only` (the three sections) and shows `.section--project-only` (the new banner + Cost by Branch table) when `body.project-mode` is set; reverse otherwise.
- Banner shows the project name + `× All projects` button → calls `setSelectedProject(null)`.

**Click handlers:**
- Wrap the project cell in Cost by Project table and Cost by Project & Branch table with `<a href="#project=<enc>" class="proj-link">…</a>`. The hash change is the source of truth; `popstate`/`hashchange` listener calls `setSelectedProject(...)`. (No inline `onclick` plumbing.)
- The Top Projects chart uses Chart.js `onClick` callback → reads clicked bar index → `setSelectedProject(top[index].project)`.
- The sessions table's project cell becomes a `.proj-link` as well.

**`setSelectedProject(name)`:**
- Sets state, writes URL hash via `history.pushState`, calls `applyFilter()`. Idempotent.

### New "Cost by Branch" table (project mode only)

Same columns as Cost by Project but with no Project column: `Branch | Sessions | Turns | Input | Output | Est. Cost`. Sortable, paginated, CSV-exportable — reusing the existing project-branch sort state and pagination, just suppressing the project column when in project mode (simpler than a parallel implementation).

Empty branches render as `(no branch)`.

## Edge cases

- **Project name with special URL chars** (spaces, `/`, `#`): always `encodeURIComponent` when writing, `decodeURIComponent` when reading. Project names in the DB come from `project_name_from_cwd(cwd)` and can contain any path component characters.
- **Stale project hash** (`#project=foo` for a project that no longer exists or has no data in range): render the empty-state banner and zero charts as usual; the `× All projects` button still works. Don't auto-clear — the user might have just narrowed the date range.
- **Project filter + model filter both active**: filters compose (AND). If the result is empty, sections render with their existing empty states; the banner remains so the user can exit project mode without losing context.
- **CSV exports in project mode**: the project + project-branch CSV exports are hidden (the sections are hidden). The other CSVs (sessions, daily) export the *filtered* (project-scoped) data, matching what's on screen. The new Cost by Branch table gets its own CSV export button mirroring the existing one.
- **Series hide state**: `hiddenSeries.project` is irrelevant in project mode (chart hidden) — leave the Set alone, it'll be reused if the user exits.

## What this does **not** do

- No server-side filter parameter — all filtering stays client-side, consistent with the existing range/model filter pattern.
- No multi-project selection. A single project is selected or not.
- No changes to the CLI or to `scanner.py` / DB schema.
- No new dependencies.

## Testing

- `tests/test_dashboard.py`: extend the existing `/api/data` test to assert each `daily_by_model_project` row contains a `project` key, and that the same `(day, model)` group can appear with different projects.
- No JS unit tests in the repo today; manual verification with the dev dashboard:
  1. Run `python cli.py dashboard`.
  2. Click a project in Top Projects chart → URL has `#project=...`, banner appears, three project sections gone, daily/hourly/model totals shrink.
  3. Refresh → still in project mode.
  4. Browser back → returns to all-projects.
  5. Click `× All projects` → hash clears, back in all-projects.
  6. Apply a date range and a model filter in project mode → composes correctly.
  7. Click a session row's project link → enters that project.

## Files touched

- [dashboard.py](dashboard.py) — `get_dashboard_data` (2 query edits + payload keys) and `HTML_TEMPLATE` (state, URL, filter, banner, branch table, click handlers, CSS).
- [tests/test_dashboard.py](tests/test_dashboard.py) — assertion update.
- [CHANGELOG.md](CHANGELOG.md) — add bullet under the current TBD heading on DEV.

No new files.
