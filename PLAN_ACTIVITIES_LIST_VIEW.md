# PLAN: Synced Activities List View

## Summary

Add `GET /activities` — a paginated, sortable, sport-filterable list of all
synced (`downloaded_exercise`) rows. This is a read-only view layered on top
of the existing `downloaded_exercise` table; no schema migration is needed
since every column the view needs (`exercise_id`, `file_path`, `sport`,
`start_time`, `downloaded_at`) already exists.

This plan was produced by spec-writer and approved by the user, including a
follow-up revision adding pagination/sort/filter. It is implemented exactly
as specified below.

---

## 1. `src/polar_fit_sync/db.py` changes

Add module-level allow-list constants (near the top, alongside
`OAUTH_STATE_TTL_SECONDS`):

```python
LIST_SORT_COLUMNS = {
    "start_time": "start_time",
    "sport": "sport",
    "downloaded_at": "downloaded_at",
}
LIST_SORT_DEFAULT = "start_time"
LIST_DIR_DEFAULT = "desc"
```

Add `distinct_sports()` — returns the distinct non-NULL sport values present
in `downloaded_exercise`, sorted, powering the activities filter dropdown so
options always reflect data actually synced.

Extend the existing `count_downloaded()` to accept an optional `sport`
param, keeping the existing no-arg call site(s) working unchanged.

Add `list_downloaded()` immediately after `count_downloaded()` — returns one
page of `downloaded_exercise` rows as plain dicts. `sort`/`direction` are
validated against fixed allow-lists (`LIST_SORT_COLUMNS` keys / `asc`,`desc`)
and a `ValueError` is raised on anything else. This is a defense-in-depth
backstop — the web layer normalizes user input to valid values before
calling — not a user-facing error path. `exercise_id ASC` is always appended
as a deterministic tiebreaker so OFFSET-based paging is stable.

**SQL-safety note**: the only interpolated strings are the resolved column
name and normalized direction, and both originate strictly from the fixed
`LIST_SORT_COLUMNS` dict / literal `("asc","desc")` tuple — never raw user
input.

## 2. `src/polar_fit_sync/web.py` changes

Add `GET /activities` inside `create_app`, near the existing `index` route.

**Critical**: `page`, `page_size`, `sport`, `sort`, `dir` are declared as
plain `str` query params (NOT typed `int`), with all numeric parsing/
clamping done manually in the handler body via try/except. Typing them as
`int` in the signature would let FastAPI/Pydantic auto-reject malformed
input with a 422 before normalization runs — but malformed/malicious query
params must always render HTTP 200 with graceful fallbacks.

Normalization order: validate `sort`/`dir`/`sport` against allow-lists
first, then parse/clamp `page_size` (1–100, default 25), compute `total`
and `total_pages` from the (already sport-filtered) count, then parse/clamp
`page` against `total_pages`. Normalization happens before rendering so no
unnormalized value leaks into the rendered links (this is asserted by the
invalid-dir-falls-back-to-default test, which requires byte-identical
output to the canonical default).

Import `LIST_SORT_COLUMNS` from `polar_fit_sync.db` alongside the existing
`Db` import.

## 3. `src/polar_fit_sync/templates/activities.html` — new file

Mirrors the inline-style convention of `index.html` (reuses `.card`/
`.label`/`.value`/`.btn`/`.btn-secondary`/`.pill`), widens
`body { max-width: 960px }`, and adds table styling.

Content:
- `<h1>Synced Activities</h1>` + "Back to status" link (`.btn-secondary`) to `/`.
- GET `<form action="/activities">` with a sport `<select>` ("All" =
  `value="all"`, then one option per entry in the full `sports` list),
  hidden inputs preserving `sort`/`dir`/`page_size` (page is dropped so
  filtering resets to page 1).
- Count line showing `{{ total }}`.
- Sortable table: Start time / Sport / Downloaded at columns toggle
  asc/desc via links, showing a `▲`/`▼` indicator on the active column;
  plus a plain "File" column.
- Row rendering: `start_time`/`sport`/`downloaded_at` fall back to `—` when
  falsy; `sport` wrapped in `.pill` when present; `file_path` styled with
  `overflow-wrap:anywhere`.
- Empty state: a message containing "No activities" when `activities` is
  empty (zero rows in DB, or a filter matching nothing).
- Pagination footer: "Page {{ page }} of {{ total_pages }}", prev/next
  links (present only when `has_prev`/`has_next`) preserving `page_size`,
  `sort`, `dir`, `sport`.

Relies on Jinja2's default autoescaping for `file_path`/`exercise_id`.

## 4. `src/polar_fit_sync/templates/index.html` — nav link

Inside the `{% if token %}` connected-card block, add a link to
`/activities` (`.btn-secondary`), matching existing spacing/style.

## 5. Tests

`tests/test_db.py`: import `LIST_SORT_COLUMNS`; append tests covering the
sort allow-list, empty-DB behavior, row shape, default sort order, stable
pagination tiebreaker, sort-by-sport asc/desc, sort-by-downloaded_at,
invalid sort/direction raising `ValueError`, sport filtering (incl.
combined with pagination), `count_downloaded` sport filter and backward
compatibility, and `distinct_sports` NULL-exclusion/dedup/sort.

`tests/test_web.py`: add a `_seed_activities` helper near `_store_token`;
append tests covering the empty-state message, populated rows, pagination
next/prev presence and clamping, invalid `page_size` fallback and max-100
clamp, sort param reflected in row order, sort/dir injection neutralized,
invalid `dir` falling back byte-identical to the default, sport filter
narrowing results, data-driven sport dropdown, unknown-sport empty result,
and the index page linking to `/activities` when a token is stored.

## Critical constraints

- Check existing imports/signatures in `db.py`/`web.py` before editing —
  merge in, don't blindly overwrite.
- `page`/`page_size`/`sort`/`dir` route params MUST be typed `str`.
- Sport filter dropdown must list ALL distinct sports in the DB, not just
  those in the current filtered/paginated result set.
- Do not modify `pyproject.toml`'s hatch packaging table.
- Run `pytest -q` after implementing and fix any failures found.
