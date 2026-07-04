# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude (Anthropic) as my AI tool throughout this project for codebase navigation, 
code explanation, and debugging guidance.

**Use 1 — Initial code reading and pattern recognition:**
I shared the full contents of all five service files with Claude and asked it to read 
through them. Claude identified suspicious patterns in each file — the `songs[:-1]` 
slice in playlist_service.py, the `weekday() != 6` condition in streak_service.py, 
the missing `create_notification()` call in notification_service.py, the 24-hour 
threshold in feed_service.py, and the `outerjoin` without `.distinct()` in 
search_service.py. This helped me know where to look before reproducing each bug. 
However I still had to reproduce every bug myself with curl commands and direct 
database queries before writing any fixes — Claude could point to suspicious code 
but could not confirm the bug without me testing it.

**Use 2 — Understanding Python datetime behavior:**
For Bug 1 (streak reset on Sundays), I asked Claude to explain exactly what 
`datetime.weekday()` returns for each day of the week. Claude explained that 
`weekday()` returns 0 for Monday through 6 for Sunday — which is why the condition 
`today.weekday() != 6` was False every Sunday and blocked the streak increment. 
I verified this by running `grep -n "weekday" services/streak_service.py` myself 
to confirm the exact line before writing the root cause analysis.

**Use 3 — Navigating unfamiliar Flask and SQLite tooling:**
When I ran into issues with the Flask shell indentation errors and could not get 
test data IDs, Claude suggested using `sqlite3` directly from the command line 
instead. That approach worked immediately and gave me all the user, song, and 
playlist IDs I needed to reproduce each bug with real data. This saved significant 
time and taught me a useful debugging technique — going directly to the database 
when the application layer is not cooperating.

**Where I verified and course-corrected:**
For Bug 3 (search duplicates), Claude predicted the duplicates would be visible 
in the API response as repeated song objects. When I tested the API, only 1 result 
appeared per song — SQLAlchemy's ORM was deduplicating at the object level. I had 
to run the raw SQL query directly against the database to prove the duplicate rows 
existed at the query level. Claude's explanation of why the outerjoin produces 
duplicate rows was accurate, but its prediction about how the bug would appear 
in the API required me to investigate further and correct through my own testing.

I also caught an error where Claude stated the date was July 4th when it was 
actually July 3rd — a small but important detail when reasoning about the 
Sunday streak bug which is date-dependent.
---

## Codebase Map

### Main Files and Their Roles

**app.py**
The Flask application factory. Creates and configures the Flask app, initializes the SQLAlchemy database connection, and registers all route blueprints. This is the entry point for the entire application.

**models.py**
Defines all five SQLAlchemy database models: User, Song, Playlist, Tag, Rating, ListeningEvent, and Notification. Also defines three association tables — friendships (user-to-user many-to-many), song_tags (song-to-tag many-to-many), and playlist_entries (playlist-to-song many-to-many with an explicit position column for ordering). Every piece of data in the app flows through these models.

**routes/songs.py**
Handles HTTP endpoints for song-related actions — searching songs, viewing a song's details, rating a song, and recording a listening event. Routes do input parsing and response formatting only. All business logic is delegated to service functions.

**routes/playlists.py**
Handles HTTP endpoints for playlist actions — creating playlists and adding songs to playlists.

**routes/users.py**
Handles HTTP endpoints for user profiles, listening streaks, and notifications.

**routes/feed.py**
Handles HTTP endpoints for the Friends Listening Now feed and the general activity feed.

**services/streak_service.py**
Contains the logic for recording listening events and updating a user's listening streak. A streak increments when a user listens on consecutive calendar days and resets to 1 if a day is skipped.

**services/feed_service.py**
Contains two functions — get_friends_listening_now() which returns friends who have listened recently within a time threshold, and get_activity_feed() which returns the most recent N listening events from all friends regardless of recency.

**services/search_service.py**
Contains search_songs() which queries songs by title or artist using a case-insensitive LIKE match, and get_song() which retrieves a single song by ID.

**services/notification_service.py**
Contains logic for creating notifications and retrieving them. Notifications are generated when friends interact with a user's shared songs — for example when someone adds their song to a playlist or rates it.

**services/playlist_service.py**
Contains logic for creating playlists, retrieving playlist metadata, and retrieving the ordered list of songs in a playlist. Songs in a playlist have an explicit position column that determines their display order.

**seed_data.py**
Populates the database with test data — 5 users, 13 songs, 3 playlists, and 10 tags — for development and testing.

---

### Data Flow — How Rating a Song Triggers a Notification

This traces the complete path of a user rating a song from HTTP request to database:

1. User sends `POST /songs/<song_id>/rate` with `user_id` and `score` in the request body
2. `routes/songs.py` receives the request, validates that `user_id` and `score` are present, and calls `notification_service.rate_song(user_id, song_id, score)`
3. `notification_service.rate_song()` validates the score is between 1 and 5, looks up the Song and User from the database, checks if a Rating already exists for this user-song pair, and either creates a new Rating or updates the existing one
4. The Rating is saved to the database via `db.session.commit()`
5. The completed Rating object is returned up through `routes/songs.py` which serializes it with `.to_dict()` and returns a 201 JSON response

---

### Patterns I Noticed

Every route immediately delegates to a service function. Routes handle only input parsing and response formatting — no business logic lives in the routes layer. All database queries and business rules live in the services layer. This separation makes the bugs easy to locate — if something is broken in an endpoint, the cause is always in the corresponding service file, not the route.

The app uses UUIDs as primary keys for all models, generated by Python's uuid module rather than auto-incrementing integers.

The playlist_entries association table is the most complex data structure — it adds a position column and an added_by column to the many-to-many relationship between playlists and songs, which is how playlist ordering is tracked.


---

## Bug Fix Root Cause Analyses

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
Queried the Late Night Vibes playlist via `GET /playlists/c21124d9-64ab-4b25-a9cd-ce3a01adf8ec/songs` and received 6 songs with `count: 6`. Then queried the database directly with `SELECT song_id, position FROM playlist_entries WHERE playlist_id='c21124d9...' ORDER BY position` and confirmed 7 songs exist in positions 1 through 7. The API was missing the song at position 7 — Free Throws by Hoop Dreams.

**How I found the root cause:**
The README confirmed bugs live in the services layer. I opened `services/playlist_service.py` and read `get_playlist_songs()`. The function queries songs ordered by position correctly, but the final return statement caught my attention immediately — it was slicing the results list before returning it.

**The root cause:**
In `get_playlist_songs()` in `services/playlist_service.py`, the return statement used Python slice notation `songs[:-1]` which means "all elements except the last one." This caused the last song in every playlist to be silently dropped from the response regardless of playlist size or content. The bug affected every playlist in the app consistently — not a conditional edge case.

**The fix and side-effect check:**
Changed `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]` — removing the erroneous slice. Verified the fix by querying all three playlists (Late Night Vibes, Friday Energy, Study Mode) and confirming each returned the correct number of songs matching the database. The `get_playlist()` and `get_user_playlists()` functions in the same file were not affected since they do not use the songs query.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**
Ran a raw SQL query directly against the database:
`SELECT s.title, s.id FROM song s LEFT JOIN song_tags st ON s.id = st.song_id WHERE s.title LIKE '%After Hours%'`
This returned 3 identical rows for After Hours — one duplicate per tag. Confirmed the same pattern for Crown Heights Anthem (3 tags, 3 duplicate rows) and Harlem Renaissance (3 tags, 3 duplicate rows). The API was masking the duplicates through SQLAlchemy's ORM object mapping but the underlying query was producing duplicate rows for every song with multiple tags.

**How I found the root cause:**
The README confirmed bugs live in the services layer. I opened `services/search_service.py` and read `search_songs()`. The function uses an `outerjoin` on the `song_tags` association table to enable tag-based filtering. I recognized that a LEFT JOIN on a many-to-many association table produces one row per matching join — so a song with 3 tags produces 3 rows. I confirmed this by running the raw SQL directly against the database which showed exactly 3 duplicate rows per multi-tag song.

**The root cause:**
In `search_songs()` in `services/search_service.py`, the query uses `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` to join songs with their tags. When a song has multiple tags this join produces one result row per tag — a song with 3 tags appears 3 times in the raw query results. SQLAlchemy's ORM deduplicates these at the object level in some configurations, but the underlying SQL query is incorrect and would produce visible duplicates under different ORM settings or with raw query execution.

**The fix and side-effect check:**
Added `.distinct()` before `.all()` in the query chain to eliminate duplicate rows at the SQL level. Verified the fix by searching for "after hours" which previously produced 3 duplicate rows in raw SQL — the API now correctly returns 1 result. Also verified that songs without tags (like Midnight Drive) still return correctly with 1 result, and that songs with a single tag (like Block Party) also return correctly — confirming `.distinct()` did not affect results for non-duplicated cases.


---

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**
Checked nova's notifications before any rating — 1 notification existed (song_added_to_playlist). Then had darius rate nova's Midnight Drive song via `POST /songs/e171822f.../rate` with score 5. Checked nova's notifications again — still 1 notification, count unchanged. The rating was saved successfully but no notification was created.

**How I found the root cause:**
The README confirmed bugs live in the services layer. I opened `services/notification_service.py` and read both `add_to_playlist()` and `rate_song()` side by side. In `add_to_playlist()` I found a `create_notification()` call after the song is added. I then read through the entire `rate_song()` function and found it ends with `db.session.commit()` and `return rating` with no notification call anywhere. The pattern used in `add_to_playlist()` was completely absent from `rate_song()`.

**The root cause:**
In `rate_song()` in `services/notification_service.py`, the function correctly saves the rating to the database but never calls `create_notification()`. The `add_to_playlist()` function in the same file follows the correct pattern — after performing its action it checks if the song's sharer is different from the acting user and creates a notification. That pattern was simply never implemented in `rate_song()`, so rating a song produced no notification under any circumstances.

**The fix and side-effect check:**
Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, mirroring the exact pattern used in `add_to_playlist()`. The condition `if song.shared_by != user_id` ensures users do not get notified when they rate their own songs. Verified the fix by having darius rate nova's Still Waters song — nova's notification count went from 1 to 2 and the new notification showed type `song_rated` with body "darius rated your song 'Still Waters' 4/5." Also confirmed that the `add_to_playlist()` notification still works correctly and was not affected by the change.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:**
Called `GET /feed/54c67a3e.../listening-now` and received 3 friends in the feed. Then queried the database directly with `SELECT user_id, song_id, listened_at FROM listening_event ORDER BY listened_at DESC` and confirmed the most recent listening events were from hours ago — not within any reasonable definition of "now." The feed was showing darius who listened at 23:02, simone at 22:57, and kenji at 22:52 on July 3rd, even though the current time was early July 4th.

**How I found the root cause:**
The README confirmed bugs live in the services layer. I opened `services/feed_service.py` and read `get_friends_listening_now()`. At the top of the file I found `RECENT_THRESHOLD = timedelta(hours=24)`. The function calculates a cutoff as `datetime.now(timezone.utc) - RECENT_THRESHOLD` and returns all friends who listened after that cutoff. A 24 hour window means anyone who listened in the past day appears as "listening now" — which is far too wide for a feature called "Friends Listening Now."

**The root cause:**
In `feed_service.py`, the constant `RECENT_THRESHOLD = timedelta(hours=24)` defines what counts as "recent" for the Friends Listening Now feed. A 24 hour threshold means the feed shows listening activity from the entire previous day as if it were happening right now. The feature name and user expectation is that "listening now" means currently active — within minutes, not hours. A friend who listened 23 hours ago is not listening now.

**The fix and side-effect check:**
Changed `RECENT_THRESHOLD = timedelta(hours=24)` to `RECENT_THRESHOLD = timedelta(minutes=30)` — 30 minutes is a reasonable window for "currently listening." Verified the fix by calling the endpoint again — the feed correctly returned empty since no friends had listened within the last 30 minutes, proving the threshold is now enforced properly. Checked `get_activity_feed()` in the same file — it does not use `RECENT_THRESHOLD` at all and was not affected by the change.

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**
Ran `grep -n "weekday" services/streak_service.py` which revealed line 73 contains `and today.weekday() != 6` as part of the streak increment condition. Python's `weekday()` returns 6 for Sunday, meaning this condition evaluates to False every Sunday — blocking the streak increment and falling through to the else branch which resets the streak to 1. Any user who listened on Saturday and then listened again on Sunday would have their streak reset despite listening on consecutive days.

**How I found the root cause:**
The README confirmed bugs live in the services layer and identified `streak_service.py` as the affected file. I opened it and read `update_listening_streak()`. The function correctly calculates `days_since_last` as the number of days between today and the last listening date. The increment condition on line 73 should fire when `days_since_last == 1` — meaning the user listened yesterday. However the condition had an extra clause `and today.weekday() != 6` which I immediately recognized as a day-of-week check. Since `weekday()` returns 6 for Sunday, this extra clause caused the entire increment branch to be skipped every Sunday.

**The root cause:**
In `update_listening_streak()` in `services/streak_service.py`, line 73 reads `elif days_since_last == 1 and today.weekday() != 6`. The `and today.weekday() != 6` condition was intended to detect a week boundary but is logically wrong. `weekday()` returns 6 for Sunday — so this condition is False every Sunday, meaning the streak never increments on Sundays regardless of whether the user listened on consecutive days. Any user who listened Saturday and then Sunday would hit the else branch and have their streak reset to 1 instead of incremented. The streak should increment whenever `days_since_last == 1` regardless of which day of the week it is.

**The fix and side-effect check:**
Removed `and today.weekday() != 6` from line 73, leaving just `elif days_since_last == 1:`. This correctly increments the streak for any consecutive day including Sunday. Verified the fix by having nova record
