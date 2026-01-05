# Spotify Data Analysis Project<br>

# Step 1: Import Libraries<br>
import pandas as pd<br>
import numpy as np<br>
import matplotlib.pyplot as plt<br>
import seaborn as sns<br>
import sqlalchemy<br>
import pymysql<br>
import spotipy<br>
from spotipy.oauth2 import SpotifyClientCredentials<br>
import re<br>

print("✅ All libraries imported successfully!")<br>

# Step 2: Spotify API Authentication<br>
client_id = 'YOUR_CLIENT_ID'<br>
client_secret = 'YOUR_CLIENT_SECRET'<br>

sp = spotipy.Spotify(auth_manager=SpotifyClientCredentials(<br>
    client_id=client_id,<br>
    client_secret=client_secret<br>
))<br>

print("✅ Spotify API authenticated successfully!")<br>

# Step 3: Collect Single Track Data<br>
track_url = "https://open.spotify.com/track/3n3Ppam7vgaVa1iaRUc9Lp"<br>
track_id = re.search(r'track/([a-zA-Z0-9]+)', track_url).group(1)<br>
track = sp.track(track_id)<br>

track_data = {<br>
    'Track Name': track['name'],<br>
    'Artist': track['artists'][0]['name'],<br>
    'Album': track['album']['name'],<br>
    'Popularity': track['popularity'],<br>
    'Duration (minutes)': track['duration_ms'] / 60000<br>
}<br>

df_track = pd.DataFrame([track_data])<br>
print("\n✅ Single Track Data:")<br>
print(df_track)<br>

# Step 4: Collect Playlist Tracks<br>
def get_playlist_tracks(sp, playlist_id):<br>
    tracks_data = []<br>
    results = sp.playlist_tracks(playlist_id, limit=100)<br>
    tracks = results['items']<br>

    while results['next']:<br>
        results = sp.next(results)<br>
        tracks.extend(results['items'])<br>

    for item in tracks:<br>
        track = item['track']<br>
        if track:<br>
            tracks_data.append({<br>
                'Track Name': track.get('name'),<br>
                'Artist': track['artists'][0].get('name') if track.get('artists') else None,<br>
                'Album': track['album'].get('name') if track.get('album') else None,<br>
                'Release Date': track['album'].get('release_date') if track.get('album') else None,<br>
                'Duration (minutes)': track.get('duration_ms') / 60000 if track.get('duration_ms') else None,<br>
                'Popularity': track.get('popularity'),<br>
                'Track ID': track.get('id')<br>
            })<br>
    return tracks_data<br>

playlist_id = '6Mf614QiAuop5ud9x5beBS'<br>
tracks_data = get_playlist_tracks(sp, playlist_id)<br>
df_playlist = pd.DataFrame(tracks_data)<br>
print(f"\n✅ Collected {len(df_playlist)} tracks from playlist!")<br>

# Step 5: Data Cleaning & Transformation<br>
df_playlist.drop_duplicates(subset='Track ID', inplace=True)<br>
df_playlist.fillna({'Popularity': 0, 'Duration (minutes)': 0}, inplace=True)<br>
df_playlist['Release Date'] = pd.to_datetime(df_playlist['Release Date'], errors='coerce')<br>

print("\n✅ Cleaned Playlist Data:")<br>
print(df_playlist.head())<br>

# Step 6: Load Data into SQL<br>
engine = sqlalchemy.create_engine("mysql+pymysql://username:password@localhost/spotify_db")<br>
df_playlist.to_sql('spotify_tracks', con=engine, if_exists='replace', index=False)<br>
print("\n✅ Data loaded into SQL successfully!")<br>

# Step 7: Analytical SQL Queries<br>
query_top_artists = """<br>
SELECT Artist, COUNT(`Track ID`) AS Total_Tracks, AVG(Popularity) AS Avg_Popularity<br>
FROM spotify_tracks<br>
GROUP BY Artist<br>
ORDER BY Avg_Popularity DESC<br>
LIMIT 10;<br>
"""<br>
top_artists = pd.read_sql(query_top_artists, engine)<br>
print("\n✅ Top Artists by Popularity:")<br>
print(top_artists)<br>

# Step 8: Visualizations<br>
plt.figure(figsize=(10,6))<br>
sns.barplot(data=top_artists, x='Artist', y='Avg_Popularity', palette='viridis')<br>
plt.xticks(rotation=45)<br>
plt.title('Top 10 Artists by Popularity')<br>
plt.ylabel('Average Popularity')<br>
plt.show()<br>

plt.figure(figsize=(10,6))<br>
sns.histplot(df_playlist['Duration (minutes)'], bins=30, kde=True, color='skyblue')<br>
plt.title('Distribution of Track Duration')<br>
plt.xlabel('Duration (minutes)')<br>
plt.show()<br>

# Step 9: Save to CSV<br>
df_playlist.to_csv('spotify_playlist_data.csv', index=False)<br>
df_track.to_csv('spotify_single_track.csv', index=False)<br>
print("\n✅ Data saved to CSV files successfully!")<br>
