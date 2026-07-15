## AI Usage
I used Claude throughout this project for a few specific things:

**Orientation:** Before touching any review comments, I had Claude walk me through 
services/collection_service.py and tests/test_collection.py to understand the existing 
add_to_collection() pattern (exception classes, dedup check ordering, fixture structure) so I could 
follow the same conventions in watchlist_service.py rather than inventing my own style.

**Verification, not authorship:** For Comment 2 (deduplication), I wrote the dedup check myself — 
Claude pointed me to the pattern in add_to_collection() to model it after, and reviewed my drafts 
for bugs (I initially tried comparing a Film object against a list of dicts from get_watchlist(), 
which would never match, and imported a nonexistent app.errors module before realizing exception 
classes are defined locally in the service file, same as collection_service.py).

**Stress-testing my design reasoning (Comments 4 and 5):** For both design-decision comments, I 
wrote my own position first, then used Claude to pressure-test it. For Comment 4 (default 
visibility), I initially reasoned that watchlists should be private because "the other list isn't 
public either" — Claude pointed out that CollectionEntry actually has no public field at all, so 
that comparison didn't hold, which pushed me to ground my argument in something specific to what a 
watchlist actually represents (an unfiltered, aspirational list) rather than borrowing logic from a 
different model. For Comment 5 (sort order), I already leaned toward agreeing with the maintainer's 
preference for date-added order, and used Claude to sharpen the "engagement with reviewer's point" 
section — specifically pushing me to name exactly when alphabetical order is actually useful (locating 
a known title) rather than dismissing it outright.

**Git troubleshooting:** During the interactive rebase (Milestone 4), I used Claude to walk through 
Vim commands (I don't normally use Vim) and to interpret a "local changes would be overwritten" error 
mid-rebase — it turned out to be a transient issue that resolved on retry with git rebase --continue, 
verified by checking git status showed a clean working tree.

My final reasoning in Comments 4 and 5 is my own — Claude never wrote the arguments, it flagged gaps 
(like the CollectionEntry comparison not holding up) and I revised from there.


## Comment 1 — Rename
**What I did:** Renamed save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py 
to match the project's verb_to_noun convention (same pattern as add_to_collection()). Updated the 
one call site in routes/watchlist/watchlist.py — both the import statement and the function call 
inside add_film().
**How I verified:** Ran grep -rn "save_to_watchlist" . across the whole repo to confirm no 
references remained (only a stale __pycache__ binary matched, which isn't source and is gitignored). 
Ran the full test suite (pytest tests/ -v) — all 4 tests still passed.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to add_to_watchlist() following the same pattern as 
add_to_collection() — query WatchlistEntry filtered by user_id and film_id, and if a match exists, 
raise a new WatchlistEntryAlreadyExistsError instead of creating a second entry.
**How I verified:** Ran the full test suite (pytest tests/ -v) — all 4 existing tests still passed. 
No existing test exercises the duplicate path directly since it's new behavior, but the check mirrors 
add_to_collection's verified logic exactly.


## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py and wrote test_add_to_watchlist_nonexistent_film_raises, 
modeled directly on test_add_to_collection_nonexistent_film_raises from test_collection.py. Reused the 
app and sample_user fixtures. Used an integer fake film_id (999999) rather than a fake UUID string, since 
Film.id is still db.Integer on this branch (pre-UUID-refactor).
**How I verified:** Ran pytest tests/test_watchlist.py -v — passed. Ran the full suite (pytest tests/ -v) 
to confirm no regressions in the collection tests.

## Comment 4 — Default visibility
**My position:** `WatchlistEntry.public` should default to `False`.

**Reasoning:** A watchlist isn't a record of what you've actually watched and formed an opinion on since it's a running list of impulses: things you added on a whim, guilty pleasures, or films you'll lose interest in before ever watching. That makes it more revealing and more exposing to judgment than a finished collection, since nothing on it has been "vetted" by the act of actually watching it. Defaulting to private means the majority of users, who never touch the setting, aren't unknowingly exposing that unfiltered list. Sharing should be something a user opts into deliberately, not something they discover by accident.

**Tradeoff acknowledged:** Defaulting private means CineLog loses the passive social/discovery value a visible watchlist could offer — friends browsing what you're excited to watch, or new users discovering trending films through others' lists. That's a real cost. But users who want that can still opt in per-entry using the existing `public` field, or just tell friends directly what they're planning to watch — the app just isn't making that disclosure decision for them by default.


## Comment 5 — Sort order
**My position:** Sort `get_watchlist()` by `date_added` (newest first), matching the maintainer's preference.

**Reasoning:** A watchlist isn't just a static list of titles — it implicitly tracks the order you decided you wanted to see things, which for most people roughly tracks the order they'd actually want to watch them in. Newest-first surfaces what you were most recently excited about, which is more useful day-to-day than an alphabetical list that scrambles that intent.

**Engagement with reviewer's point:** The maintainer's reasoning — "most users want to see what they added recently" — holds up. Alphabetical is really only better for one specific case: hunting for a known title in a long list, which a search/filter feature would solve better than the default sort order anyway. Date-added also brings watchlist behavior in line with `get_collection()`, which already sorts by `date_added.desc()` — consistent sort behavior across the app is a small but real win for anyone using both features.


## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` and `git rebase origin/main`. Two conflicts came up. 
First, `.gitignore` — main already had its own `.gitignore` since I forked, and mine (with 
`.pytest_cache/` added) didn't match main's, so it showed as an add/add conflict. Second, and the 
real conflict from the review comment: `models.py`. While my branch was open, `main` was refactored 
so `Film.id` changed from `db.Integer` to a UUID string (`db.String(36)`). My `WatchlistEntry` model 
still had `film_id = db.Column(db.Integer, db.ForeignKey("film.id"), ...)`, referencing the old type.

**How I resolved it:** For `.gitignore`, I merged both versions into one list containing every 
ignored pattern from each side. For `models.py`, I kept my `WatchlistEntry` class (since main doesn't 
have a watchlist feature at all) but changed `film_id` from `db.Integer` to `db.String(36)` so it 
matches the new `Film.id` type. After the rebase completed, I also caught two leftover spots that 
still assumed integer IDs even though they didn't block the rebase itself: `tests/test_watchlist.py` 
used `fake_film_id = 999999` (an integer) for its nonexistent-film test, which I changed to a fake 
UUID string (`"00000000-0000-0000-0000-000000000000"`) matching the pattern already used in 
`test_collection.py`. I also updated a stale docstring in `add_to_watchlist()` that still described 
`film_id` as `(int)` pre-refactor.

**How I verified no conflict remains:** After `git rebase --continue` through all commits, git 
reported "Successfully rebased and updated refs/heads/feature/watchlist." with no remaining conflict 
markers. I confirmed with `grep -n "<<<<<<<\|=======\|>>>>>>>" models.py` (empty output) and ran the 
full test suite (`pytest tests/ -v`) — all 5 tests passed both immediately after the rebase and again 
after fixing the stale integer test ID and docstring.



## PR Description

### What this feature does
Adds a watchlist feature to CineLog, letting users save films they want to watch later, separate
from their collection of films they've already watched. Includes endpoints to add a film to a
user's watchlist and view a user's full watchlist.

### Design decisions
- **Default visibility:** `WatchlistEntry.public` defaults to `False`. A watchlist reflects
unfiltered, aspirational intent (things added on a whim, not yet "vetted" by actually watching them),
so it defaults to private and requires users to opt in to sharing rather than opting out.
- **Sort order:** `get_watchlist()` sorts by `date_added` descending (newest first), matching the
maintainer's preference and bringing it in line with `get_collection()`'s existing sort behavior.
Alphabetical order is better suited to a search/filter feature than to the default view.

### How to manually test
1. Start the app: `python app.py`
2. Create a user and a film in the database (via existing setup/seed data or direct DB insert).
3. Add a film to the watchlist — `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }`
4. View the watchlist — `GET /watchlist/<user_id>` — confirm the new entry appears, sorted
   newest-first if you add more than one.
5. Attempt to add the same film again and confirm a `WatchlistEntryAlreadyExistsError` is raised
   (duplicate protection).
6. Attempt to add a nonexistent `film_id` and confirm a `FilmNotFoundError` is raised.
7. Run the automated test suite to confirm everything passes: `pytest tests/ -v`