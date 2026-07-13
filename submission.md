# Map

app.py: 

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