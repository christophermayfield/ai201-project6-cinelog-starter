# PR Response Doc — CineLog Watchlist Feature

This document responds to the six review comments from `@dev-lead`, records the
design decisions requested in the review, and includes the PR description.

---

## AI Usage

I used Cursor (Composer) in these specific ways:

1. **Codebase orientation.** Asked for a summary of `models.py`,
   `services/collection_service.py`, `routes/collection.py`, and
   `tests/test_collection.py` so I could mirror existing patterns
   (`verb_to_noun` naming, duplicate checks, HTTP status mapping, test fixtures)
   before changing watchlist code. I verified each summary against the source.
2. **Commit format check.** Ran `git log --oneline` on the original messy branch
   and asked whether the messages followed Conventional Commits and whether any
   commit bundled multiple logical changes. The AI correctly flagged
   `added watchlist model and endpoint fixed a bug more changes` as non-compliant
   and as bundling multiple changes. I confirmed that against `CONTRIBUTING.md`
   and rewrote history into separate conventional commits.
3. **Comment 4 & 5 design discussion.** I drafted my own positions first (private
   default; newest-first sort), then asked the AI for counterarguments a careful
   reviewer would raise. For visibility, that pushed me to explicitly acknowledge
   the community-discovery tradeoff and keep an opt-in `public` field on add.
   For sort order, that surfaced the long-list findability case for alphabetical
   sort, which I addressed by treating findability as a search concern rather than
   changing the default sort.

What the AI did **not** do: make the design decisions. Visibility default and
sort order are my calls, grounded in CineLog's collection-vs-watchlist semantics.

---

## Comment 1 — Rename `save_to_watchlist()` → `add_to_watchlist()`

**What I did:**
Implemented the service function as `add_to_watchlist()` from the start (after
rebasing onto `main`), and wired the route import/call site to that name. There
is no remaining `save_to_watchlist` reference in the tree.

**How I verified:**
`rg "save_to_watchlist" --glob '*.py'` returns no matches. The add endpoint
imports and calls `add_to_watchlist`.

**Why:** `CONTRIBUTING.md` requires `verb_to_noun`, and the sibling collection
API is `add_to_collection()`. Matching that keeps the two services symmetrical.

---

## Comment 2 — Deduplication when a film is already on the watchlist

**What I did:**
Mirrored `add_to_collection()`:
- Added `AlreadyInWatchlistError`.
- In `add_to_watchlist()`, after the film-exists check, query for an existing
  `(user_id, film_id)` row and raise if found.
- Added `UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")`
  on `WatchlistEntry` as a database safety net.
- Mapped `AlreadyInWatchlistError` → HTTP 409 and `FilmNotFoundError` → HTTP 404
  in the add route (same pattern as `routes/collection.py`).

**How I verified:**
`test_add_to_watchlist_duplicate_raises` asserts the second add raises and that
the row count stays at 1. Manual check: add the same film twice via the service
and confirm the exception.

---

## Comment 3 — Test for nonexistent `film_id`

**What I did:**
Added `tests/test_watchlist.py` modeled on `tests/test_collection.py`. The
directly requested case is `test_add_to_watchlist_nonexistent_film_raises`, which
uses a fake UUID and asserts `FilmNotFoundError`. Per `CONTRIBUTING.md`, I also
added happy-path, duplicate, sort-order, and default-visibility tests.

**How I verified:**
`pytest tests/ -v` → **9 passed** (4 collection + 5 watchlist).

---

## Comment 4 — Default visibility (`public`)

**My position:** Default to **private** (`public=False`).

**Reasoning:** A collection is retrospective (films already watched). A watchlist
broadcasts *intent*, which is more revealing (politics, health-adjacent titles,
etc.). Users who want a private "save for later" queue will not hunt for a
privacy toggle, so the default should protect them. Opting in to share is one
field (`"public": true` on add); opting out after accidental exposure cannot
truly un-share what others already saw.

**Tradeoff:** CineLog is a community app, and private-by-default reduces ambient
discovery ("what friends want to watch"). I accept that cost and mitigate it with
a low-friction opt-in on the same add request rather than a buried settings page.

**AI influence:** I wrote this position first, then asked the AI for a reviewer's
counterargument. It emphasized that private-by-default can fight a community
product's discovery goals. That did not change the default, but it made me
explicitly document the tradeoff and keep the opt-in `public` parameter as
mitigation.

Documented in the PR description below as well.

---

## Comment 5 — Sort order (date added vs alphabetical)

**Decision:** Implement the maintainer's preference — sort by **date added,
newest first** (`WatchlistEntry.date_added.desc()`), matching `get_collection()`.

**User behavior I'm optimizing for:**
The common watchlist loop is: save a film → later open the list to pick what to
watch next (or confirm what you just queued). That is a **recency / queue**
behavior, not a catalog lookup. Newest-first puts the film you just added at the
top (immediate feedback that the save worked) and surfaces recent intent first
when deciding "what's next?" It also matches how users already experience
`/collection`, so switching between the two lists does not require a second
mental model of ordering.

**Tradeoff of the other option (alphabetical):**
Sorting by `Film.title.asc()` would make it easier to **find a known title** in a
long list (scan to "B" for *Blade Runner*). We give that up as the default. For
a growing watchlist, A–Z buries the film you just saved somewhere mid-alphabet
and answers a less common question ("where is title X?") better than the more
common one ("what did I add / what should I watch?"). Findability is still a
real need — I treat it as a search/filter problem for a later iteration, not as
a reason to keep alphabetical as the global default. A `?sort=title|date` query
param would reclaim that option without changing the default; I deferred that as
premature until we see demand.

**What I changed:** `get_watchlist()` now orders by `date_added` descending.
Covered by `test_get_watchlist_returns_newest_first`.

---

## Comment 6 — Rebase onto `main` after UUID film ID migration

### What conflicted

Two incompatible histories of film identity met during the rebase:

1. **Type mismatch (integer vs UUID).** The original `feature/watchlist` branch
   treated film IDs as integers end-to-end:
   - `Film.id = db.Column(db.Integer, ...)`
   - `WatchlistEntry.film_id = db.Column(db.Integer, ForeignKey("film.id"))`
   - `CollectionEntry.film_id` still integer on that branch tip
   - Service docstring: `film_id (int)` with note `pre-refactor`
   - Route contract: `Body: { "film_id": <int> }`
   Meanwhile `main` already contained
   `refactor: migrate film IDs from integer to UUID` (`07ca580`), so
   `Film.id` / `CollectionEntry.film_id` on `main` were `db.String(36)`.

2. **Silent model loss on rebase.** Replaying the feature branch onto UUID
   `main` did not always surface a clean conflict marker in `models.py`. The
   UUID refactor and the watchlist addition touched the same region of the
   file; the auto-merge resolution could keep `main`'s version of that region
   and **drop the entire `WatchlistEntry` class**. The service still did
   `from models import WatchlistEntry`, so the conflict showed up as an
   `ImportError` / broken app rather than an obvious merge conflict.

### How I resolved it

1. **Fetch updated main:** `git fetch origin main`.
2. **Rebase onto the UUID base:** `git rebase` onto the post-UUID `main`
   lineage (`07ca580` → `bbe206c`), keeping history **linear**. I explicitly
   avoided leaving the branch tip on a merge commit from `main` (e.g.
   `Merge pull request #1...`), which would have reintroduced merge commits.
3. **Restore and migrate `WatchlistEntry`:** Re-added the model on top of
   UUID `main` with:
   - `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))`
   - `watchlist_entries` relationships on `User` and `Film`
   - unique constraint on `(user_id, film_id)`
4. **Update all call sites / docs to UUID:**
   - `services/watchlist_service.py`: `film_id (str): UUID of the film`
   - `routes/watchlist/watchlist.py`: `Body: { "film_id": "<uuid>" }`
   - `tests/test_watchlist.py`: UUID fixtures; nonexistent case uses
     `"00000000-0000-0000-0000-000000000000"`

### How I confirmed the conflict was fully resolved

| Check | Command / action | Expected result |
| --- | --- | --- |
| No integer film-ID leftovers | `rg 'film_id \\(int\\)\|<int\>|Integer.*ForeignKey\\("film' models.py services/watchlist_service.py routes/watchlist/ tests/test_watchlist.py` | No matches |
| Model column type | Inspect `WatchlistEntry.film_id` in `models.py` | `db.String(36)` FK to `film.id` |
| UUID refactor is in history | `git merge-base --is-ancestor 07ca580 HEAD` | Exit 0 (ancestor) |
| No merge commits on branch | `git log --merges --oneline bbe206c..HEAD` | Empty |
| App imports | `python -c "from app import create_app; create_app()"` | No `ImportError` for `WatchlistEntry` |
| Behavior with UUIDs | `pytest tests/ -v` (incl. nonexistent-film UUID test) | All tests pass |

All of the above passed. The integer/UUID conflict is fully resolved: watchlist
code matches `main`'s UUID film IDs, `WatchlistEntry` is present, and the
branch history remains linear with no merge commits.

---

## Git History (`git log --oneline`)

Verified against Conventional Commits / `CONTRIBUTING.md`: each message uses a
type prefix, imperative mood, and one logical change. No merge commits on the
branch (`git log --merges bbe206c..HEAD` is empty).

```
$ git log --oneline bbe206c..HEAD
fe827ab docs: expand Comment 6 conflict resolution process
529ee01 docs: document Comment 6 UUID rebase verification
0acf911 docs: clarify Comment 5 sort-order rationale in pr-response
27c142c docs: add pr-response.md with review responses
16dcfd0 test: add watchlist service tests
156599b feat: add watchlist service and API endpoints
eb9b8ab feat: add WatchlistEntry model with UUID film ids
```

Full recent log (includes base commits from `main`):

```
$ git log --oneline -9
fe827ab docs: expand Comment 6 conflict resolution process
529ee01 docs: document Comment 6 UUID rebase verification
0acf911 docs: clarify Comment 5 sort-order rationale in pr-response
27c142c docs: add pr-response.md with review responses
16dcfd0 test: add watchlist service tests
156599b feat: add watchlist service and API endpoints
eb9b8ab feat: add WatchlistEntry model with UUID film ids
bbe206c Merge pull request #2 from ascherj/chore/add-gitignore
718a9a8 chore: add .gitignore for generated files
```

---

## PR Description

### Overview

This PR adds a **watchlist** feature to CineLog: users can save films they want
to watch later, separately from their watched collection. It exposes
`GET /watchlist/<user_id>` (list entries) and `POST /watchlist/<user_id>/add`
(add a film by UUID), with service-layer validation for missing films and
duplicates.

### Design decisions

1. **Visibility default:** New watchlist entries default to **`public=False`**
   (private). Watchlists express future intent, which is more sensitive than a
   watched-history collection, so sharing is opt-in via an optional `"public": true`
   field on add.
2. **Sort order:** Watchlists return entries by **date added, newest first**,
   matching the collection API and matching how users typically scan a "up next"
   queue.

### Manual testing

1. Install and start the app:
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
2. Create a user and a film in the SQLite DB (or reuse seeds/fixtures), noting
   their UUID `id` values. Example using a short Flask shell / Python snippet
   with `create_app()` and `db.session`.
3. Add a film to the watchlist:
   ```bash
   curl -s -X POST http://localhost:5000/watchlist/<USER_UUID>/add \
     -H 'Content-Type: application/json' \
     -d '{"film_id":"<FILM_UUID>"}'
   ```
   Expect `201` and `"public": false`.
4. List the watchlist:
   ```bash
   curl -s http://localhost:5000/watchlist/<USER_UUID>
   ```
   Expect the film you just added at the top (newest first).
5. Add the same film again — expect `409` conflict.
6. Add a nonexistent film UUID — expect `404`.
7. Optionally add with `"public": true` and confirm the entry is public.
8. Run automated tests:
   ```bash
   pytest tests/ -v
   ```
