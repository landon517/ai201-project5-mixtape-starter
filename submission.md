# Project 5 Submission — Mixtape Bug Hunt
 
**Name:** Landon Ward 
 
## AI Usage
 
I used Claude throughout as a tool for the environment setup and some general codebase navigation.
 
**Environment setup:** I had some issues with Flask setup that I spent about 30 minutes working out with Claude. I messed up the initial environment and was confused on what web port.
 
**Codebase navigation:** Once the app was running, I used Claude to help me understand the structure, specifically how tables like `playlist_entries` and `song_tags` work. I pasted the full contents of `models.py` and all five service files and asked Claude to explain the flow.
 
**Reproduction:** Claude helped me construct the curl commands as well.
 
**Bug investigation:** I read the relevant service file myself first. For Bug #4, I found that `rate_song()` had no notification call by comparing it to `add_to_playlist()` myself. For Bug #3, I had noticed the join looked suspicious but needed help understanding the SQL behavior.
 
**Where I verified things myself:** Claude initially suggested Bug #3 would show obvious duplicates in the response, but my reproduction run returned `count: 1` with no duplicates visible. Of course I valued the reproduction over Claude's analysis.
  
---
 
## Codebase Map
 
### `app.py`
Defines `create_app()` which configures the database connection, initializes SQLAlchemy, and registers the four route blueprints (`songs`, `playlists`, `users`, `feed`) with their URL prefixes. Also sets up the database tables via `db.create_all()` on startup.
 
### `models.py`
Defines 6 SQLAlchemy models and 3 association tables:
 
- **User** — stores username, email, listening streak, and last listened timestamp. Has a self-referential many-to-many relationship with itself via the `friendships` table to represent mutual friend connections.
- **Song** — stores title, artist, album, genre, and a reference to the user who shared it
- **Tag** — simple label model with just an id and name. 
- **ListeningEvent** — a record that a specific user listened to a specific song at a specific time. 
- **Rating** — a user's 1–5 score for a song. Enforces a unique constraint so one user can only rate a song once.
- **Playlist** — a named collection of songs created by a user. Songs are linked via the `playlist_entries` association table, which adds a `position` column (integer) and an `added_by` column — meaning the order of songs in a playlist is explicitly stored, not inferred from insertion order.
- **Notification** — a message sent to a user when a friend interacts with their shared song. Has a `read` boolean flag.
### `routes/`
 
- **`songs.py`** — search songs (`GET /songs/search?q=`), get a single song (`GET /songs/<id>`), rate a song (`POST /songs/<id>/rate`), record a listen (`POST /songs/<id>/listen`)
- **`playlists.py`** — create a playlist, add a song to a playlist, get songs in a playlist
- **`users.py`** — get a user profile, get a user's streak, get a user's notifications
- **`feed.py`** — get friends listening now, get activity feed
### `services/`
The business logic layer. Each file handles one domain:
 
- **`search_service.py`** — queries the database for songs matching a title or artist string using `ilike` (case-insensitive). Uses an `outerjoin` with `song_tags` to include tag data.
- **`streak_service.py`** — calculates whether a listening event continues or resets a user's streak by comparing today's date to `last_listened_at`. Streak increments if the user listened yesterday, resets if more than one day has passed, and stays the same if they already listened today.
- **`feed_service.py`** — queries `ListeningEvent` records for a user's friends within a 24-hour window, deduplicates by friend (only most recent song per friend), and returns the result ordered by recency.
- **`notification_service.py`** — creates `Notification` records when songs are interacted with. Also owns the `rate_song` and `add_to_playlist` logic since those actions trigger notifications.
- **`playlist_service.py`** — creates playlists and retrieves songs from a playlist ordered by their `position` in `playlist_entries`.

 
## Data Flow: User Rates a Song
 
1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id": "...", "score": 4 }` in the request body.
2. `routes/songs.py` parses the JSON, validates that `user_id` and `score` are present, and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` in `notification_service.py`:
   - Validates the score is between 1 and 5.
   - Looks up the `Song` and `User` to confirm they exist.
   - Checks if a `Rating` already exists for this user/song pair. If so, updates the score; otherwise creates a new `Rating` record.
   - Commits to the database and returns the `Rating`.
4. The route returns the rating as JSON with a `201` status code.
Note: unlike `add_to_playlist`, `rate_song` does **not** create a notification for the song's sharer — this is one of the bugs in the project.
 
---
 
## Patterns I Noticed
 
- **Routes are thin, services are thick.** Every route immediately delegates to a service. The route's only job is input validation and JSON formatting. This makes the service layer easy to test in isolation (the test files import services directly, not routes).
- **Association tables carry extra data.** Both `playlist_entries` (position, added_by, added_at) and `friendships` go beyond simple many-to-many linking. This is a deliberate design choice to store relational metadata.
- **UUIDs everywhere.** Every model uses a string UUID as its primary key rather than an auto-incrementing integer. This is common in apps where IDs might be generated client-side or synced across systems.
 
## Bug Fixes
 
### Bug #3 — The same song keeps showing up twice in search
**File:** `services/search_service.py`
 
**How I reproduced it:**
`GET /songs/search?q=crown` returned `count: 1` for "Crown Heights Anthem", a song with 3 tags (rap, hip-hop, boom bap). The count was correct in this run but the underlying query was structurally broken — the `outerjoin` with `song_tags` produces one row per tag per song, meaning a song with 3 tags generates 3 rows in the result set, which causes duplicates.
 
**How I found the root cause:**
Looked at the `search_songs` function in `search_service.py`. The query joined `Song` with `song_tags` via an `outerjoin`. In SQL, joining a song to its tags multiplies the song's rows by the number of tags it has. SQLAlchemy's ORM sometimes deduplicates these automatically, sometimes doesn't — making the bug inconsistent depending on how tags are loaded.
 
**Root cause:**
The `outerjoin` on `song_tags` was unnecessary. The `Song` model already has a `tags` relationship defined in `models.py` (`tags = db.relationship("Tag", secondary=song_tags, lazy="subquery")`), which means SQLAlchemy loads tags automatically when `song.to_dict()` is called. The join was redundant and introduced the possibility of duplicate rows — one per tag per song.
 
**Fix and side-effect check:**
Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line from the query. The filter and result handling stayed identical. Verified that "Crown Heights Anthem" still returns with all 3 tags in the response after the fix, confirming the `tags` relationship loads them correctly without the join.
 
**Commit:** `e3636e0` — `fix: remove song_tags outerjoin from search query to prevent duplicate results`
 
---
 
### Bug #4 — No notification when a song is rated
**File:** `services/notification_service.py`
 
**How I reproduced it:**
`POST /songs/6da6a96e-.../rate` with `user_id: darius` and `score: 4` returned a 201 and created the rating successfully. Then `GET /users/49989502-.../notifications` (nova, the song's sharer) returned only a pre-existing playlist notification from seed data — no notification for the rating.
 
**How I found the root cause:**
Traced from `POST /songs/<id>/rate` in `routes/songs.py` → `notification_service.rate_song()`. Read the full function and compared it to `add_to_playlist()` in the same file. `add_to_playlist()` ends with a `create_notification()` call guarded by a check that the adder isn't the sharer. `rate_song()` had no equivalent call — it saved the rating and returned, nothing else.
 
**Root cause:**
`rate_song()` was never wired up to send a notification. The `create_notification()` helper existed and was used correctly in `add_to_playlist()`, but whoever wrote `rate_song()` simply omitted the notification step. The rating was saved correctly — only the notification was missing.
 
**Fix and side-effect check:**
Added a `create_notification()` call right before `return rating`, guarded by `if song.shared_by != user_id` so a user rating their own song doesn't trigger a notification to themselves. Verified by rating a song as darius and confirming nova received a new `song_rated` notification. Checked that rating your own song produces no notification, and that the rating itself still saves correctly in both cases.
  
 
### Bug #5 — The last song in a playlist never shows up
**File:** `services/playlist_service.py`
 
**How I reproduced it:**
`GET /playlists/947aef69-.../songs` returned `count: 6` and 6 songs. A direct database query (`len(p.songs)`) showed 7 songs in that playlist. The last song is always missing from the API response.
 
**How I found the root cause:**
Traced from the route `GET /playlists/<id>/songs` → `routes/playlists.py` → `playlist_service.get_playlist_songs()`. Read the return statement at the bottom of that function and immediately spotted the slice.
 
**Root cause:**
The return statement used `songs[:-1]` instead of `songs`. In Python, `[:-1]` is a slice meaning "every element except the last one." So no matter how many songs were in the playlist, the last one was always dropped before being returned. The query itself was correct — it fetched all songs in the right order — but the slice threw away the final result every time.
 
**Fix and side-effect check:**
Changed `songs[:-1]` to `songs` on the return line. No other code in the file was affected. returned 7 songs matching the database count. Checked that song order (by position) was still correct in the response.
 
