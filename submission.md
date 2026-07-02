# Submission

## AI Usage :


## Main Files :

The models.py defines the SQLAlchemy models: User, Song, Playlist, Rating, ListeningEvent, Notification, and Tag. These models are essentially classes, each being a database table. One example of these database tables is for Song which has the columns id, title, artist, albulm, and genre wish the information being stored in the rows. Some notable things to mention is that the playlist_entries join table wont just link playlists to songs but it adds a position column. This position column adds a explicit order, not simply an insertion order. It also records the added_by and added_at information. Song also as a shared_by key pointing to the user who shared it. 

 Then there is routes/ with the files songs.py, playlists.py, users.py,feed.py. These essentially will recieve the web requests, it will validate the input in the request, it will call a specific service, and then returns JSON. At this level there isn't any logic, the logic will instead live in the service/. 
 
 There is also service/ which has playlist_service.py, feed_service.py, streak_service.py, search_service.py, notification_service.py. These files will do the actual work and talk to the models, taking the URL requests and completing the logic.It has python functions that has no web knowledge. They simply do the work and talk to the database. 
 
 Finally the app.py essentially builds the app. This app is configuring the database and then registers the blueprints under a URL prefix. 

**Data Flow :**
"POST /playlists/<id>/songs → routes/playlists.py → add_to_playlist()
 → create_notification() → DB (playlist_entries row + notification row) → 201"
Essentially there is a post that requests the system to add a song to a playlist which is represented as: "POST /playlists/<id>/songs with {song_id, added_by: Person}.
Following this the route will call add_song(), validate the input, and then call add_to_playlist(). Then in service/ the function add_to_playlist() is completed adding a row and filling in the information. It checks who the song is added by and if it was added by another user then it calls create_notification(). Then a notification row is created and commited. The route will then return 201 and later on "GET /users/<Other_Person>/notifications" shows the notifiation to all the users but the one who added the song to the playlist. 

**Patterns :**
The main pattern here is that there are three-layers. The routes will handle the web, the services will handle all the logic, and the models will handle all the data. This keeps the system very organized buy also helps make the flow of requests from each section very clear and easy understand. Also errors will flow via the ValueError, meaning that services will raises a ValueError call. The routes will wrap this call in the try/except ValueError and converts it to a 404 or 400 signal. Also there is to_dict() in every model so that the JSON shape is defined once on each model. 

## Bug Reproduction :

Bug #3: "The same song keeps showing up twice in search" -> inconsistent duplicates
How I reproduced it: Ran python seed_data.py to load the database, started the app, and called GET /songs/search?q=Vrown Heights. The song "Crown Heights Anthem" came back three times. I then reashced for "Block Party" and "Midnight Dive" each of which only appeared one time. Then number of duplicates matchd the numner of tags on each respective song. It's important to note that songs with no tag will automatically appear once if their name is searched. 

Bug #5: "The last song in a playlist never shows up" -> Missing entry
How I reproduced it: I ran pytest/tests/test_playlists.py. This creates a playlists with 5 songs with the positions going from 1 to 5. Then test_playlists_returns_all_songs() has failed since all 4 out of the 5 songs were returned. When checking the file playlists_service.py it's shown that it will return [songs.to_dict() for song in songs [:-1]]. This "[:-1]" leads to the last list element being excluded as the list is sorted by position. 

Bug #2: "Friends Listening Now shows people from yesterday" -> Feed updating improperly
How I reproduced it: I ran python seed_data.py and then called get_friends_listening_now() for the user kenji. It returned nova as "listening now" but the listening event was timestaped to 2 hours ago. This means that nova isn't currently listening and the "listening now" list is incorrect. The root cause seems to be that get_friends_listning_now() is using the RECENT_THRESHOLD = timedelta(hours=24) as its cutoff. This makes it so that anyone who has listened to music in the past 24 hours will be shown as currently listening to music but that isn't the case. The threshold should be smaller window to truly accomplish a "listening now" feed. 

## Root Cause Analysis Entries :

Bug #3:

Navigation Path:
I started at GET /songs/search?q=... which goes to routes/songs.py, then that calls search_songs() in services/search_service.py. That's where I found the query with the .outerjoin on song_tags.

Analysis:
The root cause is the query joins the song_tags table onto Song. When you join like that you get one row back for every match, so if a song has 3 tags it matches 3 rows in song_tags and comes back as the same song 3 times. Then the code just does [song.to_dict() for song in results] with no way to remove the copies, so all 3 show up in the response. That's why the number of duplicates always matched the number of tags, and why a song with no tags only showed up once. The annoying part is the join wasn't even needed, because to_dict() already grabs the tags through the model relationship. So the join was doing nothing except making duplicates.

My fix was to just delete the .outerjoin(song_tags, ...) line from the query. Nothing else needs it, the title/artist search still works and to_dict() still fills in the tags on its own, so now each song only comes back once. After that I checked that songs still show their tags list, that the search is still case insensitive, and that songs with no tags still appear. I reran tests/test_search.py to make sure each song only shows up one time now.

Bug #5:

Navigation Path:
I started at GET /playlists/<id>/songs in routes/playlists.py, which calls get_playlist_songs() in services/playlist_service.py. The problem was on the return line: [song.to_dict() for song in songs[:-1]].

Analysis:
The query itself is fine, it pulls every song in the playlist ordered by position. The problem is the [:-1] on the return line. In Python [:-1] gives you everything except the last item, and since the list is sorted by position the last item is always the highest position song, so the actual last song in the playlist. So the query grabbed all 5 songs and then the [:-1] just threw away song 5 right before sending it back. It wasn't an ordering or counting problem in the query at all, it was only the [:-1] cutting off the end.

My fix was to change songs[:-1] to just songs so it returns the whole list. The slice wasn't doing anything useful, the docstring even says the function returns all songs, so taking it out gives the behavior it was supposed to have. Afterwards I checked that the order is still right (the order_by position is untouched), that an empty playlist still returns [] (it did before too), and that a playlist with only one song now returns that song instead of an empty list. I reran tests/test_playlists.py and test_playlists_returns_all_songs() passes now with all 5 songs.

Bug #2:

Navigation Path:
I started at the friends listening now feed, GET /users/<id>/feed in routes/feed.py, which calls get_friends_listening_now() in services/feed_service.py. The important part is the RECENT_THRESHOLD = timedelta(hours=24) constant it uses to build the cutoff.

Analysis:
The way the function decides who is "listening now" is it takes cutoff = now - RECENT_THRESHOLD and keeps any event newer than that. But RECENT_THRESHOLD is set to 24 hours, so anyone who listened to anything in the last 24 hours counts as listening right now. The query and the dedupe logic are actually fine, the real issue is the window is way too big for what "listening now" is supposed to mean. Nova's event was 2 hours ago which is easily inside 24 hours, so she got shown as listening now even though she wasn't. For a real "listening now" feed the window should only be a few minutes.

My fix was to change RECENT_THRESHOLD to a short window, timedelta(minutes=5), so only people who actually listened just now show up. The comparison and the ordering stay the same, the only thing that changes is how big the window is, which is the one thing causing the bug. Afterwards I checked that get_activity_feed() isn't affected, because it doesn't use RECENT_THRESHOLD at all (its docstring says it returns the most recent events no matter when they happened). I also checked the timezone comparison still lines up (both sides are UTC), and that a friend with no recent listen now drops off the list while someone listening in the last few minutes still shows up.





  