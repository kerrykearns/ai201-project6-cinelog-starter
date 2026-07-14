# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
Renamed save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py to match the project's verb_to_noun naming convention used elsewhere (e.g. add_to_collection() in collection_service.py). Updated the import and call site in routes/watchlist/watchlist.py to match.
**How I verified:**
Ran a project-wide search (ripgrep, no filters) for the literal string "save_to_watchlist" before and after the change. Before: 3 matches (the function definition, the import, and the call site in the POST /watchlist/<user_id>/add route). After: 0 matches. Also ran the full test suite (pytest tests/ -v) — all 4 existing tests passed, confirming the rename didn't break anything.

## Comment 2 — Deduplication
**What I did:**
Added an AlreadyInWatchlistError exception (mirroring AlreadyInCollectionError's minimal style) and a deduplication check in add_to_watchlist(). After the existing FilmNotFoundError check, the function now queries WatchlistEntry for an existing row matching (user_id, film_id) and raises AlreadyInWatchlistError if one is found, before creating and committing the new entry.
**How I verified:**
Reviewed add_to_collection()'s existing deduplication logic in services/collection_service.py before implementing — it uses the same check-then-raise pattern (query by (user_id, film_id), raise a specific exception if found, insert only if the check passes). Followed this pattern exactly for consistency. Ran pytest tests/ -v to confirm the change didn't break existing tests. 

## Comment 3 — Missing test
**What I did:**
Created tests/test_watchlist.py with test_add_to_watchlist_nonexistent_film_raises, modeled directly on test_add_to_collection_nonexistent_film_raises from tests/test_collection.py. It reuses the same app and sample_user fixture structure (in-memory SQLite, same setup/teardown), the same fake-UUID approach for a nonexistent film_id, and the same with pytest.raises(FilmNotFoundError): assertion style — calling add_to_watchlist() instead of add_to_collection().
**How I verified:**
Ran pytest tests/test_watchlist.py -v to confirm the new test passes, then pytest tests/ -v to confirm the full suite (5 tests total) passes with no regressions.

## Comment 4 — Default visibility
**My position:**
Watchlists should default to public=True — but on the strength of a real argument, not just inheriting the default.

**Reasoning:**
CineLog is a film tracking app with a collection feature already built around dedup logic and a to_dict() shape clearly meant for feed/list rendering — the app's data model assumes lists are shown, not just stored. A watchlist specifically is a forward-looking signal ("I want to watch this") rather than a completed opinion, which is exactly the kind of content that drives the social/discovery loop apps like this depend on: friends see what you want to watch, get inspired, add it to their own list, and the whole thing becomes a lightweight recommendation engine without CineLog needing to build one. Defaulting private would mean the watchlist feature launches invisible by default, which undercuts the reason to build it as a separate feature from a private "notes" field in the first place.

**Tradeoff acknowledged:**
The real cost of public=True is that a watchlist can be a more sensitive signal than a finished collection — it can reveal genre interests, or gaps ("I still haven't seen X classic"), or unfinished/embarrassing intent, before the person has vetted whether they even want to be seen wanting it. A private default protects users from that exposure by default, at the cost of the feature being mostly dead social weight for most users, since most people don't proactively flip visibility toggles. Given CineLog is positioning watchlist as a companion feature to a public-by-nature collection feature, I think the discovery value outweighs the exposure risk here, but this is exactly why the codebase already exposes public as a per-entry field rather than a global setting — a per-item override, not a global privacy setting, is the safety valve for users who want to keep a specific item private.


## Comment 5 — Sort order
**My position:**
Keep date-added order — agreeing with the maintainer's preference, not alphabetical.

**Reasoning:**
A watchlist is a queue, not a reference list. The action a user takes against it is almost always "what do I feel like watching next," and the most recent additions are disproportionately the ones still top-of-mind — someone just recommended a film, or they just watched a trailer. Alphabetical order buries that recency signal under whatever the film's title starts with, which is actively unhelpful for a queue-like UI, even though alphabetical does make sense for the existing collection feature (a completed, browsable archive people might scan to find "did I already log this").

**Engagement with reviewer's point:**
The maintainer's reasoning — "most users want to see what they added recently" — is right, and matches the general UX pattern of "add to list" features elsewhere (playlists, reading lists, cart-like UIs almost universally default to recency, not alphabetical). The one place I'd push back slightly: strict date-added descending can feel chaotic if someone dumps 15 films in one sitting, since order within that batch is essentially arbitrary from the user's perspective. That's a minor concern though, and not worth adding complexity for — I'd rather ship the simple, correct default (date-added descending) than invent a hybrid sort the maintainer didn't ask for.

## Comment 6 — Rebase
**What conflicted:**
While my feature/watchlist branch was open, main merged a refactor changing Film.id (and CollectionEntry.film_id) from Integer to String(36) UUIDs. Rebasing onto origin/main surfaced two issues: (1) a trivial add/add conflict in .gitignore, since both branches independently added one, and (2) a deeper problem — my original commit that added the watchlist feature never actually included the WatchlistEntry model definition in models.py, so after rebasing onto main's rewritten models.py, services/watchlist_service.py was importing a WatchlistEntry class that no longer existed anywhere, breaking the whole test suite with an ImportError.

**How I resolved it:**
For the .gitignore conflict, I merged both versions into one deduplicated file. For the missing model, I added the WatchlistEntry class directly to models.py, written against the current UUID schema (film_id as db.String(36) with a ForeignKey to film.id, matching CollectionEntry's pattern), including a `film` relationship since get_watchlist() depends on entry.film.to_dict().

**How I verified no conflict remains:**
Ran pytest tests/ -v — all 5 tests pass. Also ran a grep for "Integer" across models.py, watchlist_service.py, and the watchlist routes to confirm no remaining integer-typed film ID references. Confirmed git log --oneline origin/main..HEAD shows a fully linear history with no merge commits.

## PR Description