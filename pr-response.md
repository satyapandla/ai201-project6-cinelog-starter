# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `verb_to_noun` naming convention used by `add_to_collection()` in the collection service. Updated the import and function call in `routes/watchlist/watchlist.py`, the only call site.
**How I verified:** Searched the codebase for all references to `save_to_watchlist` (routes and services directories) to confirm `routes/watchlist/watchlist.py` was the only call site. Ran `pytest tests/ -v` after the change to confirm the existing test suite still passed.

## Comment 2 — Deduplication
**What I did:** Added a deduplication check to `add_to_watchlist()`, following the same pattern used in `add_to_collection()`: query for an existing `WatchlistEntry` matching `user_id` and `film_id` via `.filter_by().first()` before inserting a new entry. If one exists, raise a new `AlreadyInWatchlistError` exception (mirroring `AlreadyInCollectionError`) instead of silently creating a duplicate. Also added a corresponding `except AlreadyInWatchlistError` clause in the route handler, returning a 409 status.
**How I verified:** Confirmed `WatchlistEntry` has no unique constraint on `(user_id, film_id)` in `models.py` (unlike `CollectionEntry`, which does) — meaning without this check, duplicates could actually be written to the database. Ran the full test suite to confirm no regressions.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. Used the same fixture structure (`app`, `sample_user`) and the same assertion pattern (`pytest.raises`). Used an integer fake film ID (`999999`) rather than a UUID string, since `Film.id` is still `db.Integer` on this pre-refactor branch.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` to confirm the new test passes in isolation, then ran `pytest tests/ -v` to confirm the full suite (collection + watchlist tests) passes together.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->