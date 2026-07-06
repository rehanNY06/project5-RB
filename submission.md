# Mixtape Bug Hunt — Submission

## Codebase Map

**Main files:**
- `app.py` — Flask app factory. Registers 4 blueprints (songs, playlists, users, feed).
- `models.py` — 7 entities: User, Tag, Song, ListeningEvent, Rating, Playlist, Notification.
- `routes/` — thin layer; parses requests, calls a service, formats JSON response.
- `services/` — all business logic (streak, feed, search, notifications, playlists).

**Data flow — rating a song:**
POST /songs/<id>/rate → routes/songs.py:rate() → notification_service.rate_song() → saves rating, commits. No notification is created (unlike add_to_playlist, which does notify).

**Pattern noticed:** routes never contain logic — they just call a service function and return JSON.

## Issue Plan
1. Streak resets — streak_service.py
2. Feed shows yesterday — feed_service.py
3. Duplicate search results — search_service.py
4. Missing rating notification — notification_service.py (found: rate_song never calls create_notification)
5. Last playlist song missing — playlist_service.py

Plan: start with #4 and #5, then #1.

### Issue #5: Last playlist song never shows up

**How I reproduced it:**
Queried the `playlist_entries` join table directly for playlist "Late Night Vibes" 
(id 1b15f863-...) and got a count of 7 songs. Hitting GET /playlists/<id>/songs 
returned only 6 songs — the last one in position order was missing.

**How I found the root cause:**
Read through `services/playlist_service.py`, specifically `get_playlist_songs()`. 
The function correctly queries songs joined against `playlist_entries`, ordered 
by `position` ascending. But the return line was `songs[:-1]` — slicing off the 
last element of the already-correctly-ordered list right before returning it.

**The root cause:**
`get_playlist_songs()` queries and orders the songs correctly, but the final 
line applies a `[:-1]` slice to the result list before converting to dicts. 
This unconditionally drops the last song in position order from every playlist, 
regardless of how many songs it actually has.

**My fix and side-effect check:**
Changed `songs[:-1]` to `songs` so the full ordered list is returned. Verified 
by re-checking the playlist's song count via the join table (7) against the 
API response (now also 7, previously 6). Checked `get_playlist()` and 
`get_user_playlists()` in the same file — neither uses this slicing pattern, 
so the fix is isolated to this one function.

### Issue #4: Missing notification when a friend rates your song

**How I reproduced it:**
Song "Midnight Drive" (id a75253e1-...) is shared by user ba954335-... 
Had a different user (a1feb4b4-..., "darius") rate the song via 
POST /songs/<id>/rate. The rating was saved successfully (confirmed in the 
response: new Rating record with correct user_id/song_id/score). Checked 
the sharer's notifications via GET /users/ba954335-.../notifications — 
the list still only showed the pre-existing "song_added_to_playlist" 
notification from seed data. No new notification was created for the rating.

**How I found the root cause:**
Compared `rate_song()` to `add_to_playlist()` in `notification_service.py`, since 
both functions represent a friend interacting with a user's shared song. 
`add_to_playlist()` calls `create_notification()` right after committing its 
change. `rate_song()` performs a very similar sequence — look up the song, 
save a change, commit — but never calls `create_notification()` anywhere.

**The root cause:**
`rate_song()` saves the Rating correctly but was never given the notification 
step that its sibling function `add_to_playlist()` has. The notification 
system itself works fine; it was simply never wired up to the rating flow. 
This is an omission, not a broken condition.

**My fix and side-effect check:**
Added a call to `create_notification()` at the end of `rate_song()`, guarded by 
`if song.shared_by != user_id` so a user rating their own shared song doesn't 
notify themselves — mirroring the same guard already used in 
`add_to_playlist()`. Verified by rating a song as a different user and 
confirming a new "song_rated" notification appeared for the sharer 
(count went from 1 to 2). Checked that re-rating the same song (updating 
an existing Rating) still creates a new notification each time, which seems 
reasonable since each new rating is a new event worth notifying about.