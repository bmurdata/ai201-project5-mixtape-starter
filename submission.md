# Map

app.py: 
Starts the application and reads from routes and services.

services folder:

user.py: defines functions to:
* get user information 
* Get user notifications 
* Get user streaks
* Get user read notifications

songs.py: defines functions to:
* Search songs by song name or artist name
* [POST] Allow users to rate songs
* [POST] Allow users to listen to songs

playlists.py:
POST / — Create a playlist

Requires name and created_by in the JSON body; is_collaborative defaults to True
Returns 400 if required fields are missing
Calls create_playlist(), returns the new playlist as JSON with status 201
Catches ValueError → 400 error response

GET /<playlist_id> — Get playlist details

Calls get_playlist()
Catches ValueError → 404 error response


GET /<playlist_id>/songs — List songs in a playlist

Calls get_playlist_songs(), returns songs plus a count
Catches ValueError → 404 error response


POST /<playlist_id>/songs — Add a song to a playlist

Requires song_id and added_by in the JSON body
Calls add_to_playlist() (from the notification service, interestingly — likely because adding a song also triggers a notification)
Returns 400 if required fields are missing, or if a ValueError is raised

feed.py::

GET /<user_id>/listening-now — Shows what the user's friends are currently listening to

Calls get_friends_listening_now(user_id)
Returns {"feed": [...], "count": N}
ValueError → 404 (likely if the user doesn't exist)


GET /<user_id>/activity — Returns a general activity feed for the user (probably friend activity like shares, ratings, playlist additions, etc., based on the earlier models)

Calls get_activity_feed(user_id)
Returns {"feed": [...], "count": N}
ValueError → 404


models.py: defines 7 classes 3 association tables
3 association tables (many-to-many relationships):

friendships — symmetric self-referencing user-to-user connections
song_tags — links songs to tags
playlist_entries — links playlists to songs, with extra fields for track position, who added_by the song, and added_at timestamp

7 models:

User — has username, email, a listening streak counter, and last-listened timestamp. Relates to songs they've shared, ratings they've given, listening events, notifications, playlists they created, and friends (via the self-referencing friendships table).
Tag — simple name label for categorizing songs.
Song — title, artist, album, genre, who shared it and when, plus an optional share note. Linked to ratings, listening events, and tags.
ListeningEvent — records that a specific user listened to a specific song at a given time.
Rating — a 1–5 score a user gives a song, with a unique constraint ensuring one rating per user per song.
Playlist — named collection of songs, tracks creator and creation time, and whether it's collaborative. Linked to songs through the playlist_entries table.
Notification — a message sent to a user (with type, body text, timestamp, and read/unread status).

Data Flow: search for song /songs/search?q={song name or artist}
returns songs by an artist or by name

## Issues:

### Issue 1-My listening streak keeps resetting
User: kenji
Description: Streak resets every Sunday
Steps User took:

Listened to a song every calendar day, including Saturday.
Listened again Sunday morning and checked my streak (GET /users/<my_id>/streak).
Expected: streak goes from 12 to 13 — I listened on consecutive days. Actual: streak shows 1, as if I'd skipped a day.

Steps to reproduce: 

Steps to resolve:

### Issue 2-Friends Listening Now shows people from yesterday
User: nova
9am it showed darius "listening now" to a song he told me he played at 11pm last night, 
Steps User took:

Opened my feed in the morning (GET /feed/<my_id>/listening-now).
Cross-checked with darius: his last listen was the previous night.
Expected: only friends who have listened today appear. Actual: friends whose last listen was yesterday evening still show up the next morning.

### Issue 3-The same song keeps showing up twice in search
User: simone 
I searched "Anthem" and Crown Heights Anthem by Borough Kings showed up three times in the results. Other songs only show up once.
Steps I took:

Searched for a song (GET /songs/search?q=Anthem).
Counted the results.
Expected: each matching song appears exactly once. Actual: some songs appear once, others two or three times, for a single-song match.

Steps to reproduce: 
Navigated to GET request  /songs/search?q=Anthem and every other song, including "el".  

Result: Unable to reproduce behavior. It seems the .outerjoin should be causing an error, however it does not due to .all() as documented here: https://docs.sqlalchemy.org/en/14/orm/query.html

The Query object, when asked to return either a sequence or iterator that consists of full ORM-mapped entities, will deduplicate entries based on primary key. See the FAQ for more details.

Fix: .outerjoin removed as in the future it may impact the application if SQLalchemy API changes.