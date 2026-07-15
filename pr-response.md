# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->


## Comment 1 — Rename

**What I did:** I changed the function name `save_to_watchlist` to `add_to_watchlist` to adhere to the project's naming convention of `verb_to_noun`, aligning it with parallel features like `add_to_collection()`. I performed a project-wide search and updated all corresponding call sites, including its definition in `services/watchlist_service.py` and its invocation in the REST API endpoints in `routes/watchlist/watchlist.py`.

**How I verified:** I manually verified the change by running the local Flask development server and calling the POST `/watchlist/<user_id>/add` endpoint with a test payload using `curl`. The endpoint successfully processed the request and invoked the newly renamed service function without raising any `NameError` or routing exceptions.

## Comment 2 — Deduplication
**What I did:** I looked at how `add_to_collection()` in `services/collection_service.py` handles this same problem: it queries for an existing `CollectionEntry` with the same `user_id`/`film_id` before inserting, and raises a custom `AlreadyInCollectionError` if one is found. I followed that exact pattern in `services/watchlist_service.py` — `add_to_watchlist()` now queries for an existing `WatchlistEntry` with the same `user_id` and `film_id` and raises a new `DuplicateWatchlistEntryError` if a match exists, before ever constructing or committing a new entry. I had claude implement this.

**How I verified:** I called `add_to_watchlist()` twice in a row with the same `user_id`/`film_id` in a local Python shell against the app context — the first call succeeded and returned the new entry, and the second call raised `DuplicateWatchlistEntryError` instead of creating a second row. I then verified the same behavior through the Flask test client by POSTing to `/watchlist/<user_id>/add` twice with an identical `film_id` payload: the first request returned `201`, and the second returned `409` with a descriptive error message, matching the collection endpoint's behavior for duplicates.

## Comment 3 — Missing test
**What I did:** I created a new `tests/test_watchlist.py` file and added `test_add_to_watchlist_nonexistent_film_raises`, I had claude implement this following the same fixture and assertion pattern as `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. It reuses the same `app` and `sample_user` fixtures, then asserts that calling `add_to_watchlist()` with a film_id that doesn't exist raises `FilmNotFoundError`.

**How I verified:** I ran the full test suite with `pytest` and confirmed all 5 tests pass, including the new watchlist test alongside the existing collection tests.

## Comment 4 — Default visibility
**My position:** I'm keeping `public=True` as the default on `WatchlistEntry`.

**Reasoning:** CineLog's core value proposition is community-driven discovery, not private list-keeping — the whole point of a watchlist feature on a social film-tracking app is that it feeds activity streams, lets other users see "what's next" for people they follow, and helps solve the cold-start problem every social app faces (an empty, all-private network has nothing to discover). This mirrors the convention on comparable platforms like Letterboxd, where diary entries and watchlists are public-by-default with an explicit opt-out, precisely because the product is built around social signal. We're not inventing a privacy stance from scratch here — `WatchlistEntry` already has a `public` boolean column, and `get_watchlist()` surfaces it in every returned dict (`services/watchlist_service.py`), so the opt-out mechanism already exists in the data model; this decision is just about which value we ship as the default.

**Tradeoff acknowledged:** The real cost here is a "first-run privacy surprise" — a user could add something they don't want broadcast (an embarrassing pick, a film they're screening ahead of a surprise movie night) before they've discovered the toggle exists, and by then the intent is already public. I don't think the fix is flipping the default to private, since that undermines the entire social feature for the vast majority of users who don't care and quietly kills the discovery mechanic we're building this for. The fix is making the toggle visible where it matters — surfaced at signup or the first time a user adds to their watchlist — rather than defaulting to a more private, less useful product for everyone.

## Comment 5 — Sort order
**My position:** I agree with @dev-lead — switching `get_watchlist()` in `services/watchlist_service.py` to sort by `date_added` descending (most recent first), replacing the current `Film.title.asc()` order.

**Reasoning:** A watchlist and a collection serve fundamentally different mental models. `get_collection()` (`services/collection_service.py`) is an archive — something a user browses after the fact, often to check "have I already watched this?", where alphabetical-by-title is a reasonable lookup key. A watchlist isn't an archive, it's a queue: it exists to answer "what did I just add, and what am I planning to watch next?" Alphabetical order actively works against that — if I add a film after a friend's recommendation, it gets buried between whatever else happens to share a letter, instead of surfacing at the top where I'd actually look for it.

**Engagement with reviewer's point:** @dev-lead's point about users wanting to see what they added recently is exactly right, and it's worth noting we already made this same call for `get_collection()`, which sorts by `date_added` descending for newest-first. Sorting the watchlist alphabetically was inconsistent with that precedent, not a deliberate product decision — this change brings both features in line with the same convention rather than leaving alphabetical as a one-off exception. I'm not proposing a hybrid or a sort query param for this PR; that's reasonable as a future enhancement, but @dev-lead asked for a decision, and recency-first is the correct default for how a watchlist is actually used.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->