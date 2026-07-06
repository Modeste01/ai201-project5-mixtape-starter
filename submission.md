# Project 5: Mixtape Bug Hunt Submission

## AI Usage

I used Claude Code extensively throughout this project for codebase navigation, bug investigation, and verification. Here's specifically how AI helped:

**Codebase orientation phase:**
- Asked Claude to read and summarize each service file's purpose and main functions
- Used Claude to trace data flows (e.g., "trace how a song rating triggers a notification")
- Had Claude explain the SQLAlchemy relationships in models.py and how join tables work

**Bug investigation:**
- For Bug #3 (search duplicates): Asked Claude to explain what `.outerjoin(song_tags)` does and why it might create duplicates. Claude helped me understand that LEFT OUTER JOIN creates row multiplication when a song has multiple tags. I then verified this myself by running raw SQL queries to see the actual row counts.
- For Bug #5 (missing last song): Claude helped me understand Python's slice notation `[:-1]` - I recognized it in the code but had Claude confirm that it excludes the last element.
- For Bug #2 (24h threshold): Used Claude to calculate time differences between timestamps and explain what `timedelta(hours=24)` means in the context of a "Listening Now" feature.

**What I verified myself (AI was incomplete/wrong):**
- Claude initially suggested that SQLAlchemy's `.all()` would return duplicates from the join in Bug #3, but when I tested it, SQLAlchemy actually deduplicated the Song objects automatically. I had to run raw SQL queries to prove the join was still creating row multiplication at the database level, even though the ORM masked it.
- For reproducing the bugs, I wrote my own Python test scripts rather than relying on Claude's suggestions. Claude is better at explaining code than predicting runtime behavior.

**Code generation:**
- All three bug fixes (the actual code changes) were written by me after understanding the root cause
- Claude helped draft the root cause analysis entries in this document, which I then edited for accuracy and clarity
- Git commit messages were written with Claude's help to follow conventional commit format

**Overall collaboration pattern:**
- I used Claude as a "senior developer explaining the codebase" rather than as a bug finder
- The workflow was: read code myself → ask Claude to explain what I don't understand → form hypothesis → verify by running code → fix
- Claude was most valuable for accelerating orientation in an unfamiliar codebase, not for diagnosing bugs directly

---

## Git Log

```
a96d9ee fix: change Friends Listening Now threshold from 24 hours to 1 hour
5b00e4a fix: remove unnecessary song_tags join that could cause duplicate search results
96d8ab9 fix: remove incorrect list slice that excluded last song from playlists
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```

---

## Codebase Map

### Main Files and Their Roles

**`models.py`** — Defines 7 SQLAlchemy models for all database entities:
- `User`: stores username, email, listening_streak, last_listened_at, and has relationships to songs, ratings, listening events, notifications, playlists, and friends (many-to-many via `friendships` table)
- `Song`: stores title, artist, album, genre, shared_by (foreign key to User), shared_at, and share_note
- `Rating`: stores user ratings for songs (1-5 score), with a unique constraint ensuring one rating per user-song pair
- `ListeningEvent`: records when a user listened to a song (user_id, song_id, listened_at)
- `Playlist`: stores playlist metadata (name, created_by, is_collaborative) and has a many-to-many relationship with songs via `playlist_entries`
- `Notification`: stores notifications for users (notification_type, body, created_at, read status)
- `Tag`: stores tag names that can be associated with songs via the `song_tags` join table

**Association tables**:
- `friendships`: many-to-many symmetric relationship between users
- `song_tags`: many-to-many relationship between songs and tags
- `playlist_entries`: many-to-many relationship between playlists and songs with additional columns (position, added_by, added_at) to track song order in playlists

**`app.py`** — Flask application factory that creates the app instance, initializes SQLAlchemy, and registers all route blueprints.

**`routes/`** — Flask blueprints that handle HTTP requests and delegate to service functions:
- `songs.py`: handles `/search` (song search), `/<song_id>` (song detail), `/<song_id>/rate` (rating), `/<song_id>/listen` (listening event)
- `playlists.py`: handles playlist creation and song management
- `users.py`: handles user profiles, streaks, notifications
- `feed.py`: handles friends' listening activity feed

**`services/`** — Business logic layer where all the actual functionality lives:
- `streak_service.py`: manages listening streak logic (increments on consecutive days, resets if day skipped)
- `feed_service.py`: provides "Friends Listening Now" feed (filtered by 24-hour recency) and general activity feed
- `search_service.py`: handles song search by title/artist
- `notification_service.py`: creates and retrieves notifications, plus handles the `add_to_playlist()` and `rate_song()` logic that trigger notifications
- `playlist_service.py`: creates playlists and retrieves playlist songs in order

---

### Data Flow Example: User Rates a Song

1. User sends `POST /songs/<song_id>/rate` with `user_id` and `score` in JSON body
2. Route handler `routes/songs.py:rate()` receives the request, validates inputs
3. Route calls `notification_service.rate_song(user_id, song_id, score)`
4. `rate_song()` function:
   - Validates score is 1-5
   - Looks up the song and user from database
   - Checks if user already rated this song (query Rating table filtered by user_id and song_id)
   - If existing rating found: updates the score
   - If no existing rating: creates new Rating record and adds to session
   - Commits to database
   - Returns the Rating instance
5. Route converts Rating to dict via `rating.to_dict()` and returns JSON response with 201 status

**Pattern observed**: Every route immediately delegates to a service function. Routes handle HTTP concerns (parsing request data, formatting responses, status codes). Services handle all business logic (database queries, model updates, validation). This separation makes the codebase easier to test and reason about.

---

## Bug Fixes

### Bug #1: My listening streak keeps resetting

**How I reproduced it:**
(To be completed)

**How I found the root cause:**
(To be completed)

**The root cause:**
(To be completed)

**My fix and side-effect check:**
(To be completed)

---

### Bug #2: Friends Listening Now shows people from yesterday

**How I reproduced it:**
Queried the database to check listening event timestamps - found events ranging from 0.3 hours ago to 50+ hours ago. Called `get_friends_listening_now()` for user 'nova' and examined which friends appeared. The function defines "listening now" as within the last 24 hours (RECENT_THRESHOLD constant). This means a friend who listened 23 hours ago (yesterday afternoon) would still show up in the "Listening Now" feed, which doesn't match the feature's intent. The issue title "shows people from yesterday" is accurate because 24 hours ago is literally yesterday for most users.

**How I found the root cause:**
Started at [services/feed_service.py:16](services/feed_service.py#L16) where `get_friends_listening_now()` is defined. The function uses a cutoff of `datetime.now(timezone.utc) - RECENT_THRESHOLD` at line 32. Traced back to where RECENT_THRESHOLD is defined at [line 13](services/feed_service.py#L13): `RECENT_THRESHOLD = timedelta(hours=24)`. This constant defines what "listening now" means. The docstring at line 18 says "Return a list of friends who have listened to something recently" but 24 hours is not "now" - it's "today and yesterday". For a "Listening Now" feed (real-time activity), 1 hour or less is more appropriate.

**The root cause:**
The `RECENT_THRESHOLD` constant is set to `timedelta(hours=24)`, which is far too long for a "Listening Now" feature. 24 hours means the feed shows anyone who listened at any point in the past day, including yesterday. This makes the feed stale and not useful for discovering what friends are currently listening to. The feature name implies real-time or near-real-time activity (within the last hour or so), but the implementation treats "now" as "anytime in the last 24 hours".

**My fix and side-effect check:**
Changed line 13 from `RECENT_THRESHOLD = timedelta(hours=24)` to `RECENT_THRESHOLD = timedelta(hours=1)`. This makes "Listening Now" mean "listened within the last hour" which aligns with user expectations for a real-time activity feed. Verified the fix by calling `get_friends_listening_now()` - it now returns only friends with events in the last hour. Checked that `get_activity_feed()` is unaffected since it doesn't use RECENT_THRESHOLD (it returns the most recent N events regardless of time). No side effects since RECENT_THRESHOLD is only used in the one function that has "now" in its name.

---

### Bug #5: The last song in a playlist never shows up

**How I reproduced it:**
Called `get_playlist_songs()` from `playlist_service.py` on all three seed playlists. Each playlist has 7 songs in the `playlist_entries` table (verified via direct database query), but the service function only returned 6 songs for each playlist. The last song (highest position value) was consistently missing.

**How I found the root cause:**
Started with [services/playlist_service.py:38](services/playlist_service.py#L38) which implements `get_playlist_songs()`. Traced the query logic line by line: it correctly joins Song with playlist_entries, filters by playlist_id, orders by position ascending, and calls `.all()` to get all songs. Then at [line 66](services/playlist_service.py#L66), I saw `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice notation excludes the last element of the list - this is Python's syntax for "all elements except the last one".

**The root cause:**
The return statement uses Python list slicing `songs[:-1]` which returns all songs except the last one. This was likely a copy-paste error or leftover debugging code. The slice should not be there at all - the function should return all songs in the playlist.

**My fix and side-effect check:**
Changed line 66 from `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]`. Removed the `[:-1]` slice. Verified the fix by calling `get_playlist_songs()` on all three playlists - each now correctly returns 7 songs instead of 6. No side effects expected since this only affects the return value of this one function. Other playlist-related functions like `get_playlist()` and `get_user_playlists()` don't call this function, so they're unaffected.

---

### Bug #3: The same song keeps showing up twice in search

**How I reproduced it:**
Searched for songs that have multiple tags (like "Crown Heights Anthem" which has 3 tags: rap, hip-hop, boom bap). Verified via raw SQL query that the LEFT OUTER JOIN on song_tags creates multiple rows when a song has multiple tags - for example, "Crown Heights Anthem" produces 3 rows in the join result (one per tag). While SQLAlchemy's ORM `.all()` method deduplicates these to return unique Song objects, the underlying SQL query is inefficient and creates row multiplication that could cause issues in certain query patterns or result in actual duplicates under different SQLAlchemy configurations.

**How I found the root cause:**
Started at [services/search_service.py:26](services/search_service.py#L26) in the `search_songs()` function. The query uses `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` at line 27. Traced why this join exists - looked at what data the function returns and found it returns `song.to_dict()` for each song at line 37. Checked [models.py:92-103](models.py#L92-L103) where `Song.to_dict()` is defined, and saw that it accesses `self.tags` via the relationship already defined at line 90: `tags = db.relationship("Tag", secondary=song_tags, lazy="subquery")`. This relationship already handles loading tags, so the manual join in the search query is completely redundant.

**The root cause:**
The `search_songs()` function performs an unnecessary `.outerjoin(song_tags)` that creates duplicate rows when a song has multiple tags. The join serves no purpose because the Song model already has a `tags` relationship configured with `secondary=song_tags` that loads tags automatically. When a song has N tags, the join creates N rows in the SQL result set. While SQLAlchemy's ORM deduplicates these when converting to Song objects, this is inefficient and violates the principle of querying only what you need. The join also makes the query harder to understand and maintain.

**My fix and side-effect check:**
Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line entirely from the query. The query now directly filters on Song.title and Song.artist without joining to song_tags. Tags are still accessible in the results because `Song.to_dict()` uses the relationship to load them. Verified the fix by searching for songs with 0 tags, 1 tag, and 3+ tags - all return correct results with tags still included in the output. No side effects since the join was never used (the filter doesn't reference song_tags, and tags come from the relationship, not the join).

---
