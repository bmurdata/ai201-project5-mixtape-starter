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

#### Root cause analysis
1. Issue 1-My listening streak keeps resetting
2. How you reproduced it: Checked /users/<user_id>/streak and saw streak was set to 12. Hit the songs/song_id/listen to record a listen on a Sunday, and saw streak reset to 1.
4. How you found the root cause: Checked the songs/song_id/listen endpoint and saw line 73 had a check to update the streak if the days_since_last was 1 and if today wasnt a Sunday. If it is a sunday, the streak does not update.
4. The root cause: update_listening_streak line 73 checks if it is Sunday, and if it is, does not update the streak.
5. Fix and side effect check: Removed and condition and reran browser test to confirm fix.
6. Notes: Recomit to add submission


### Issue 2-Friends Listening Now shows people from yesterday
User: nova
9am it showed darius "listening now" to a song he told me he played at 11pm last night, 
Steps User took:

Opened my feed in the morning (GET /feed/<my_id>/listening-now).
Cross-checked with darius: his last listen was the previous night.
Expected: only friends who have listened today appear. Actual: friends whose last listen was yesterday evening still show up the next morning.

#### Root cause analysis
1. Issue 2-Friends Listening Now shows people from yesterday
2. How you reproduced it: checked /feed/nova_id/listening-now, confirmed darius shows up as listening now when last listen was day before. 
4. How you found the root cause: checked feed.py to find the listening-now endpoint, and found it called get_friends_listening_now from feed_services. I then found it had a cutoff threshold of 24 hours, which is one source of the bug. The user last_listened_at is also not checked, which showed darius even though they were on the app the day before.
4. The root cause: RECENT_THRESHOLD is set to 24 hours, and is used to set a cutoff of 24 hours ago, which includes yesterday. Additionally, the for loop doesnt check the last_listened_at field from the user. 
5. Fix and side effect check: Edited get_friends_listening_now function to use a cutoff at the start of today, and added a check for user friends last_listened_at to be today as well. This removed darius. I re-ran and ensured it would work as intended.
6. Notes: Additional condition used when friend last listened is None

### Issue 3-The same song keeps showing up twice in search
User: simone 
I searched "Anthem" and Crown Heights Anthem by Borough Kings showed up three times in the results. Other songs only show up once.
Steps I took:

Searched for a song (GET /songs/search?q=Anthem).
Counted the results.
Expected: each matching song appears exactly once. Actual: some songs appear once, others two or three times, for a single-song match.
#### Root cause analysis
1. Issue 3-The same song keeps showing up twice in search
2. How you reproduced it: Navigated to GET request  /songs/search?q=Anthem and every other song, including "el".  
4. How you found the root cause: I was unable to reproduce the bug on the browser. I then checked the search_services file. There, it seems the .outerjoin should be causing an error, however it does not due to .all() as documented here: https://docs.sqlalchemy.org/en/14/orm/query.html
4. The root cause: Root cause was found to be outerjoin in /search_services file
5. Fix and side effect check: .outerjoin removed as in the future it may impact the application if SQLalchemy API changes. Re ran application and no errors found
6. Notes: The Query object, when asked to return either a sequence or iterator that consists of full ORM-mapped entities, will deduplicate entries based on primary key. See the FAQ for more details.

### Issue 4-I got notified when a friend added my song to a playlist but not when they rated it
User: aaliya 
Rating notification not working 
Notifications work when someone adds a song I shared to a playlist — I get "kenji added your song…" right away. But when kenji rated one of my songs (he showed me, 5 stars), I never got a notification. No delay, just nothing, and there's nothing in my notification list (GET /users/<my_id>/notifications) either

Steps I took:

Had a friend add my shared song to a playlist → notification arrived. ✅
Had the same friend rate a different song I shared (POST /songs/<song_id>/rate) → checked my notifications.
Expected: a notification for the rating, same as for the playlist add. Actual: rating is saved (it shows on the song), but no notification is ever created.
#### Root cause analysis
1. Issue 4-I got notified when a friend added my song to a playlist but not when they rated it
2. How you reproduced it:  Checked the GET /users/<my_id>/notifications before and after running POST /songs/<song_id>/rate and did not see a notification of the new rating. I also added a song to a playlist and saw that a notification was created.
4. How you found the root cause: Checked the notification_service for add_to_playlist and saw that it was creating a notification when a song was shared by someone else.
4. The root cause: notification not created when song rated in notification_services
5. Fix and side effect check: Updated rate_song to create a notification when a user rates a song shared by a different user by adding a create_notification call
6. Notes: 

