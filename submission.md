### DATA FLOW

`feed_service.py` has two main functions:
- `get_friends_listening_now`: returns friends who listened to something in the last 24 hours, with each friend’s most recent song.
- `get_activity_feed`: returns the most recent listening events from all friends, limited by a count.
These are called by `routes/feed.py`, which exposes `/feed/<user_id>/listening-now` and `/feed/<user_id>/activity`. `app.py` registers the feed blueprint, so those endpoints are made available.

`playlist_service.py` handles playlist logic:
- `create_playlist`: creates a new playlist record.
- `get_playlist_songs`: returns songs in a playlist ordered by their playlist position.
- `get_playlist`: returns playlist metadata.
- `get_user_playlists`: returns playlists created by a specific user.
Routes in `routes/playlists.py` call these service functions for playlist creation, detail lookup, song listing, and adding songs.

`search_service.py` handles song lookup:
- `search_songs`: searches songs by title or artist.
- `get_song`: loads a single song by ID.
`routes/songs.py` exposes these from `/songs/search` and `/songs/<song_id>`.

`streak_service.py` handles listening events and streaks:
- `record_listening_event`: creates a `ListeningEvent` record and updates the user’s streak.
- `get_streak`: returns the current streak for a user.
`routes/songs.py` calls `record_listening_event` from `POST /songs/<song_id>/listen`, and `routes/users.py` calls `get_streak` from `GET /users/<user_id>/streak`.

One real data flow:
- A client posts to `/songs/<song_id>/listen` with `user_id`.
- `routes/songs.py` calls `services.streak_service.record_listening_event(user_id, song_id)`.
- That service creates a `ListeningEvent` and updates the `User` record.
- Later, when someone requests `/feed/<user_id>/listening-now`, `routes/feed.py` calls `services.feed_service.get_friends_listening_now(user_id)`, which queries `ListeningEvent` rows for the user’s friends and returns the most recent song for each friend.

Pattern I noticed: routes mostly parse requests and return JSON; business logic stays in `services/`, and `app.py` just initializes the app and registers blueprints.

### Root Cause Analyses

1) Issue number and title: #1 — Playlist songs missing one entry when returned by service

How you reproduced it: Ran the playlist tests in `tests/test_playlists.py` which seed a playlist with 5 songs and call `get_playlist_songs(playlist_id)`; the failing behavior returned only 4 songs during local runs before fixes.

How you found the root cause: I inspected `services/playlist_service.py` and the test fixture in `tests/test_playlists.py` to verify how songs and `playlist_entries` were inserted. I followed the query in `get_playlist_songs()` and confirmed the join/filter/order logic would determine the returned rows.

The root cause: The query originally joined the wrong columns (song to playlist id) causing one playlist entry to be excluded from the join result. This meant one song row never matched and was dropped by the join.

Your fix and side-effect check: I corrected the join condition so `Song.id` is compared to `playlist_entries.c.song_id` and the `filter` uses `playlist_entries.c.playlist_id == playlist_id`. After the change I ran `pytest` (all playlist tests) to confirm all 5 songs are returned and validated `test_empty_playlist_returns_empty_list` to ensure empty playlists still return `[]`.

2) Issue number and title: #2 — Duplicate search results when songs have multiple tags

How you reproduced it: Ran `tests/test_search.py` which seeds songs with varying tag counts and calls `search_songs("Crown Heights")`; the failing behavior returned the same song multiple times (once per tag) prior to fix.

How you found the root cause: I reviewed `services/search_service.py` and the `song_tags` association table usage in the tests. I confirmed the query used an outer join to include tag rows and that multiple matching tag rows could produce repeated `Song` rows in the result set.

The root cause: The query did not deduplicate song rows produced by the join with `song_tags`, so songs with N matching tag rows appeared N times in the `.all()` result.

Your fix and side-effect check: I added a `.distinct()` call to the query (and used an `outerjoin` so songs without tags still appear). I reran `pytest` and verified `test_search_no_duplicates_multi_tag_song`, `test_search_no_duplicates_single_tag_song`, and `test_search_no_duplicates_no_tag_song` all pass. I also checked the basic search tests to ensure results still include songs that match title/artist and that songs with no tags remain discoverable.

3) Issue number and title: #3 — Listening streak was resetting incorrectly after consecutive days

How you reproduced it: I ran the streak tests in `tests/test_streaks.py` and exercised the sequence where a user listened on Saturday and then on Sunday. Before the fix, the streak reset instead of continuing to increment.

How you found the root cause: I inspected `services/streak_service.py` and followed the logic in `update_listening_streak()`. The key moment was seeing the condition that only allowed the streak to increment when `days_since_last == 1` and `today.weekday() != 6`, which blocked the Sunday case.

The root cause: The streak logic treated Sunday as a special boundary case and prevented the streak from increasing when the previous listen was on Saturday and the current listen was on Sunday. In other words, the condition incorrectly excluded the Sunday transition even though the streak should continue.

Your fix and side-effect check: I removed the `today.weekday() != 6` restriction so the streak now increments whenever `days_since_last == 1`. I verified the change with the streak tests, including the Saturday-to-Sunday scenario, and confirmed that the streak no longer resets incorrectly while still preserving the other streak rules for same-day listens and skipped days.

4) Issue number and title: #4 — Listening-now feed showed old listens from previous days

How you reproduced it: I created a scenario where a friend had a listening event from yesterday and another from today, then called `get_friends_listening_now(user_id)`; before the fix, the older event was still included because the feed was using a rolling 24-hour window.

How you found the root cause: I inspected `services/feed_service.py` and traced the logic in `get_friends_listening_now()`. The turning point was seeing the code compute a `cutoff` as `datetime.now(timezone.utc) - timedelta(hours=24)`, which made the feed behave like “last 24 hours” rather than “today’s listens.”

The root cause: The feed was filtering by a 24-hour rolling window instead of the current calendar day. That meant any listening event from the previous day could appear in “listening now” if it fell within the last 24 hours.

Your fix and side-effect check: I changed the cutoff to the start of the current UTC day (`datetime(now.year, now.month, now.day, tzinfo=timezone.utc)`) so only events from today are shown. I checked the surrounding feed behavior afterward to confirm the activity feed still returns recent events independently and the listening-now feed now reflects only today’s activity.

5) Issue number and title: #5 — Rating a song did not notify the person who shared it

How you reproduced it: I created a song shared by one user, had another user rate that song, and then checked for a notification for the original sharer. Before the fix, the rating was saved but no notification appeared.

How you found the root cause: I inspected `services/notification_service.py` and followed the `rate_song()` flow. The code committed the rating successfully, but I did not see any notification creation branch in that path, so the behavior was clearly missing at the point where the rating was processed.

The root cause: The rating path updated the `Rating` record but never called the notification mechanism for the song’s sharer. As a result, users who shared a song were not informed when someone rated it.

Your fix and side-effect check: I added a notification step in `rate_song()` that runs after the rating is committed and creates a `song_rated` notification when the rater is not the original sharer. I also verified that existing playlist notification behavior still works and that the new notification path does not create duplicate or self-notifications.
