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