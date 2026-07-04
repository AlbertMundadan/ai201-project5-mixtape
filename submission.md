## AI Usage

- I used AI tools to summarize different functions and help me troubleshoot code. For example. I did not know what today.weekday() is supposed to return so I plugged it into AI and it helped me identify the first bug. I then verified this by reporducing the issue myself. I also then suggested a fix to the AI model and ask if it meets the criteria I expect.

- I also used AI tools to help me determine the optimal way to remove duplicate song results. First I though about using distinct on the database query but the AI model suggested removing song_tags entirely since the outerjoin was not necessary to begin with.

## Codebase Map

### Entry point & wiring

- [app.py](app.py) — the app factory `create_app()`. Owns the single `db = SQLAlchemy()` instance that every other module imports. Defaults to `sqlite:///mixtape.db`, calls `db.create_all()` on startup, and registers four blueprints under URL prefixes: `/songs`, `/playlists`, `/users`, `/feed`.

### Data model — [models.py](models.py)

Seven mapped entities plus three association tables. Every primary key is a `String(36)` UUID generated in Python (`generate_uuid()`).

Core tables: **User**, **Tag**, **Song**, **ListeningEvent**, **Rating**, **Playlist**, **Notification**.

The three association tables are where the real design decisions live:

- **`friendships`** — self-referential M2M on User. Friendship is stored as _two directed rows_ (see `add_friendship` in [seed_data.py:41](seed_data.py#L41), which inserts both `(a,b)` and `(b,a)`). So `user.friends` only returns people the user explicitly points at; symmetry is a seed-data convention, not a schema guarantee.
- **`song_tags`** — plain M2M between Song and Tag.
- **`playlist_entries`** — it carries `position` (explicit ordering — songs have a rank, not just insertion order), `added_by`, and `added_at`. Playlist order is data, not arrival sequence.

- **Ratings are their own table** ([models.py:121](models.py#L121)), not a column on Song, with a `UniqueConstraint(user_id, song_id)` — a user can rate a song at most once. `rate_song()` relies on this by upserting.
- **Streak state is denormalized onto User** — `listening_streak` and `last_listened_at` are columns on User ([models.py:45](models.py#L45)), recomputed on each listen rather than derived from `ListeningEvent` rows at read time.
- Only `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification` define `to_dict()`. `Tag` has none — tags surface only as name strings inside `Song.to_dict()["tags"]`.

### Routes

- [routes/songs.py](routes/songs.py) — `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Note the rate/listen split: rating goes to `notification_service`, listening goes to `streak_service`.
- [routes/playlists.py](routes/playlists.py) — create, get metadata, `GET /<id>/songs`, `POST /<id>/songs`. Adding a song routes through `notification_service.add_to_playlist` (not `playlist_service`) because the add _also_ fires a notification.
- [routes/users.py](routes/users.py) — user profile, streak, notifications list, and `POST /users/notifications/<id>/read`.
- [routes/feed.py](routes/feed.py) — `GET /feed/<user_id>/listening-now` and `/activity`.

Every route follows the same shape: pull params, 400 on missing input, call one service function inside `try/except ValueError`, and translate `ValueError` into a 404 or 400. Services signal "not found" and "bad input" the same way — by raising `ValueError` — and routes decide the status code.

### Services

- [services/search_service.py](services/search_service.py) — `search_songs(query)` does a case-insensitive `ILIKE` on title **or** artist, `outerjoin`ed to `song_tags`. `get_song(id)` fetches one.
- [services/streak_service.py](services/streak_service.py) — `record_listening_event()` writes a `ListeningEvent` and calls `update_listening_streak()`, which compares `now.date()` to `last_listened_at.date()`: same day → no change, one day gap → increment, larger gap → reset to 1. Handles naive/aware datetime by coercing to UTC.
- [services/feed_service.py](services/feed_service.py) — `get_friends_listening_now()` fetches the user's friends, pulls their `ListeningEvent`s newer than a `RECENT_THRESHOLD`, and dedupes to the single most-recent song per friend. `get_activity_feed()` is the un-filtered sibling: most-recent N events regardless of age, no dedup.
- [services/notification_service.py](services/notification_service.py) — `create_notification()` is the shared writer. `add_to_playlist()` appends the song to the playlist and notifies the song's original sharer (unless the sharer is the adder). `rate_song()` upserts a Rating. Plus `get_notifications()` / `mark_as_read()`.
- [services/playlist_service.py](services/playlist_service.py) — `create_playlist()`, `get_playlist_songs()` (joins `playlist_entries`, orders by `position`), `get_playlist()` metadata, `get_user_playlists()`.

### Traced data flow — a user rates a song

`POST /songs/<song_id>/rate` with JSON `{user_id, score}` →
[routes/songs.py:29](routes/songs.py#L29) validates both fields are present, casts `score` to `int`, calls →
`rate_song(user_id, song_id, score)` in [services/notification_service.py:73](services/notification_service.py#L73), which bounds-checks 1–5, verifies the song and user exist (else `ValueError`), then **upserts**: if a Rating row already exists for that `(user_id, song_id)` it overwrites `score`; otherwise it inserts a new one, then commits. The route serializes `rating.to_dict()` with `201`.

### Traced data flow — viewing playlist songs

`GET /playlists/<id>/songs` → [routes/playlists.py:34](routes/playlists.py#L34) → `get_playlist_songs(id)` in [services/playlist_service.py:38](services/playlist_service.py#L38). It joins `Song` to `playlist_entries`, filters by playlist, and orders ascending by `position` — so the `position` column, not row insertion, defines what "playlist order" means end to end.

### Notable Patterns

1. **Route → single service call.** No route contains business logic beyond input parsing and status-code mapping. If an endpoint misbehaves, the cause is in the service it calls, not the route.
2. **Services own the transaction.** Every `db.session.commit()` in the app is inside a service function. Routes never commit.
3. **`ValueError` is the app-wide "expected failure" channel.** Services raise it for both missing records and invalid input; routes catch it and pick 400 vs 404. There are no custom exception types.
4. **Ordering and recency are explicit data, not implicit.** Playlist order comes from `playlist_entries.position`; feed recency comes from an explicit `RECENT_THRESHOLD` cutoff compared against `listened_at`. Neither relies on natural row/insert order.
5. **`add_to_playlist` straddles two concerns** — it mutates the playlist _and_ sends a notification — which is why it lives in `notification_service` rather than `playlist_service`, and why the playlist route imports from the notification service.

## Issues

### Issue #1 (My listening streak keeps resetting)

- Issue was reproduced using the test_streak_increments_on_sunday function in tests which tests the streak incrementation when the day is sunday.
- I found the cause by checking the streak_service.py since I knew this file handles streaks. I then checked the function that seemed to handle the operation of updating streaks which was update_listening_streak. I scanned through this function and found a condition that did not seem necessary.
- The root cause was in the update_listening_streak function where it checked if the today.weekday() != 6 before updating the streak, causing updates to be reset on sunday when weekday = 6.
- The fix was to remove this condition in the update streak function because it was unecessary since any consecutive days listened should increment the streak. The test_streaks suite was run after this change and all tests passed (including the test_streak_increments_on_sunday and other increment related tests) as expected for this minimal change.

### Issue #3 ( The same song keeps showing up twice in search)

- Issue was reproduced by searching for a song using the test_serach test suite that had multiple tags. Namely the function test_search_no_duplicates_multi_tag_song.
- I found the issue by checking the search_service.py because this file handles search functions and then finding the function that matches this path which would be search_songs. I then found the usage of song_tags despite it not being necessary.
- Root cause was in the function search_songs where it did an outerjoin onto song_tags, producing one result row per song-tag pair, so multi-tag songs duplicated.
- The fix was to remove the unnecessary join, so each matching song is returned once. This also allowed for the removal of the song_tag import. The test_search suite was ran after this fix and all tests passed to ensure there were no unintended sideffects. Searching performed as expected and test_search_no_duplicates_multi_tag_song test also passed.

### Issue #4 (I got notified when a friend added my song to a playlist but not when they rated it)

- Reproduced by having one user rate another user's shared song and checking the sharer's notifications — none were created (while playlist-adds did notify).
- I found the issue by exploring the notification_service.py file and comparing the add_to_playlist and rate_song functions to find differnces. This allowed me to notice the lack of notification creation for rate_song.
- Root cause in notification_service.py: rate_song never called create_notification, unlike add_to_playlist.
- Fix: after saving the rating, create a song_rated notification for song.shared_by, skipping the case where the rater is the sharer. This behavior of adding a notification was tested to ensure it worked as expected without sideffects. Notifications were produced for users if someone else rated it.

# Git log

afd4460 (HEAD -> bugfix/mixtape, origin/bugfix/mixtape) fix: create notification for user rating songs

784cab0 fix: remove outerjoin with tags in search_songs to prevent duplicate results

3645936 fix: streak update logic to increment on consecutive days regardless of the day of the week

2dfdeaa (origin/main, origin/HEAD, main) Add .gitignore file and update README with setup instructions

7b64551 initial commit
