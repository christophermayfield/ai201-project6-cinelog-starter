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

**My position:** Agree with the maintainer — sort by **date added, newest first**
(`WatchlistEntry.date_added.desc()`), matching `get_collection()`.

**Reasoning:** A watchlist is a queue of intent, not a reference catalog. Users
usually ask "what did I just save / what should I watch next?" Newest-first
answers that and also confirms a successful add (the new film appears at the top).
Alphabetical helps locate a known title in a long list, but that is better solved
with search/filter later than by making A–Z the global default. Consistency with
`/collection` also matters so users do not hold two mental models of ordering.

**AI influence:** After drafting, I asked the AI what counterargument a careful
reviewer would raise. It surfaced long-list findability for alphabetical sort.
I kept newest-first and addressed findability as a search concern (not a reason
to change the default). I considered `?sort=` and deferred it as premature.

Implemented in `get_watchlist()` and covered by `test_get_watchlist_returns_newest_first`.

---

## Comment 6 — Rebase onto `main` after UUID film ID migration

**What conflicted:**
`feature/watchlist` still used integer `film_id`s while `main` migrated `Film.id`
and `CollectionEntry.film_id` to UUIDs. Replaying the old branch onto `main`
would also drop `WatchlistEntry` from `models.py` (the UUID refactor on `main`
removed that class because it lived in the same region of the file).

**How I resolved it:**
- Reset the branch onto current `main` and re-applied the watchlist feature on
  top of the UUID models.
- Re-added `WatchlistEntry` with `film_id = db.Column(db.String(36), ...)` and
  relationships on `User` / `Film`.
- Updated service/route docs and payloads to use UUID strings, not integers.

**How I verified:**
- App imports cleanly; all tests pass with UUID fixtures.
- `git log --merges origin/main..HEAD` is empty (linear history, no merges).
- Branch tip sits on top of `origin/main`.

---

## Git History (`git log --oneline`)

Verified against Conventional Commits / `CONTRIBUTING.md`: each message uses a
type prefix, imperative mood, and one logical change. No merge commits on the
PR range (`git log --merges origin/main..HEAD` is empty).

```
$ git log --oneline origin/main..HEAD
7ca2951 docs: add pr-response.md with review responses
16dcfd0 test: add watchlist service tests
156599b feat: add watchlist service and API endpoints
eb9b8ab feat: add WatchlistEntry model with UUID film ids
```

Full recent log (includes base commits from `main`):

```
$ git log --oneline -8
7ca2951 docs: add pr-response.md with review responses
16dcfd0 test: add watchlist service tests
156599b feat: add watchlist service and API endpoints
eb9b8ab feat: add WatchlistEntry model with UUID film ids
bbe206c Merge pull request #2 from ascherj/chore/add-gitignore
718a9a8 chore: add .gitignore for generated files
07ca580 refactor: migrate film IDs from integer to UUID
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
