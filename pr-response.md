# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` to match the `verb_to_noun` pattern `add_to_collection()` already uses. Updated the one call site in `routes/watchlist/watchlist.py`.
**How I verified:** Searched the codebase to confirm that was the only place calling the old name. Ran the test suite — still passing.

## Comment 2 — Deduplication
**What I did:** Added a check in `add_to_watchlist()` that looks for an existing entry before inserting a new one, same as `add_to_collection()` does. If it finds one, it raises a new `AlreadyInWatchlistError` instead of silently creating a duplicate. Updated the route to catch that error and return a 409.
**How I verified:** Checked `models.py` and confirmed `WatchlistEntry` has no unique constraint on `(user_id, film_id)` — so this check is the only thing actually stopping duplicates. Ran the full test suite to make sure nothing broke.

## Comment 3 — Missing test
**What I did:** Added `tests/test_watchlist.py` with a test for adding a film that doesn't exist, modeled on the equivalent test in `test_collection.py`. Used an integer fake ID instead of a UUID, since films are still integer IDs on this branch.
**How I verified:** Ran the new test on its own, then ran the full suite to confirm everything still passes together.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.
**Reasoning:** CineLog is social by nature — people are here to see what others are watching and get recommendations. A watchlist isn't sensitive info, it's basically "movies I want to see," so defaulting it to public keeps that social feed populated without making everyone flip a setting just to use the app normally.
**Tradeoff acknowledged:** Some users won't realize their watchlist is public until it shows something they'd rather keep private. That's a real risk with any public-by-default setting. The `public` parameter we added at least lets people opt out per entry — the bigger fix would be surfacing that choice more clearly in the UI, but that's outside this PR.

## Comment 5 — Sort order
**My position:** Agree with the maintainer — sort by date added (newest first) instead of alphabetical.
**Reasoning:** A watchlist is something you're actively building, not a static list you browse — recency is what actually matters when you're deciding what to watch next.
**Engagement with reviewer's point:** This also lines up with how `get_collection()` already works, which already sorts by date added. So alphabetical was actually the odd one out, not the other way around. Didn't see a good reason to keep watchlists different from collections here, so I made the change.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` and `git rebase origin/main`. Got conflicts in `app.py` (two different import paths for the watchlist blueprint) and `services/watchlist_service.py` (an old vs. new way of fetching a film from the DB). After the rebase finished, I also noticed `models.py` still had `Film.id` as an integer, even though `main` had already switched it to a UUID — that one didn't show up as a git conflict, I just caught it by checking the file.
**How I resolved it:** Kept my branch's version for the app.py and watchlist_service.py conflicts. For the UUID mismatch, manually updated `Film.id` and the two `film_id` foreign keys in `models.py` to UUID strings, and updated my test's fake film ID to match.
**How I verified no conflict remains:** Checked for leftover `<<<<<<<` markers in the files, then ran the test suite to make sure everything still worked with the updated schema.

## PR Description

### What this feature does
Adds a watchlist to CineLog so users can save films they want to watch later, separate from their collection of films they've already watched. Includes a `WatchlistEntry` model, `add_to_watchlist()` and `get_watchlist()` service functions, and two REST endpoints:

- `GET /watchlist/<user_id>` — returns a user's watchlist
- `POST /watchlist/<user_id>/add` — adds a film to a user's watchlist

### Design decisions
- **Default visibility:** New watchlist entries default to `public=True`. CineLog is a social app built around users discovering films through each other, so a public-by-default watchlist keeps that discovery feed populated without requiring extra setup from every user. See Comment 4 in this doc for the full reasoning and tradeoffs.
- **Sort order:** `get_watchlist()` now returns entries sorted by `date_added` descending (most recent first), matching the existing pattern in `get_collection()`. A watchlist is something users are actively building, so recency is the most useful ordering. See Comment 5 for the full argument.

### How to manually test
1. Start the app: `python app.py`
2. Create a user and a film through the existing endpoints (or seed data), and note their IDs.
3. Add a film to the watchlist:
    curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add 
    -H "Content-Type: application/json" 
    -d '{"film_id": "<film_id>"}'
Should return `201` with the new entry.
4. Try adding the same film again — should return `409` (duplicate).
5. Try adding a film ID that doesn't exist — should return `404`.
6. View the watchlist:
    curl http://127.0.0.1:5000/watchlist/<user_id>
    Should return the list, newest addition first.
7. Run the automated test suite: `pytest tests/ -v` — all tests should pass.