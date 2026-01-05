# Spotify Data Analysis Project


# Step 1: Import Libraries
import pandas as pd <br>
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import sqlalchemy
import pymysql
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import re

print("All libraries imported successfully!")


# Step 2: Spotify API Authentication
client_id = 'YOUR_CLIENT_ID'      # Replace with your Spotify Client ID
client_secret = 'YOUR_CLIENT_SECRET'  # Replace with your Spotify Client Secret

sp = spotipy.Spotify(auth_manager=SpotifyClientCredentials(
    client_id=client_id,
    client_secret=client_secret
))

print("Spotify API authenticated successfully!")


# Step 3: Collect Track Data from URL
# Example: "Shape of You" by Ed Sheeran
track_url = "https://open.spotify.com/track/3n3Ppam7vgaVa1iaRUc9Lp"
track_id = re.search(r'track/([a-zA-Z0-9]+)', track_url).group(1)
track = sp.track(track_id)

track_data = {
    'Track Name': track['name'],
    'Artist': track['artists'][0]['name'],
    'Album': track['album']['name'],
    'Popularity': track['popularity'],
    'Duration (minutes)': track['duration_ms'] / 60000
}

df_track = pd.DataFrame([track_data])
print("\nTrack Data:")
print(df_track)


# ================================
# Step 4: Collect Playlist Tracks
# ================================
def get_playlist_tracks(sp, playlist_id):
    tracks_data = []
    results = sp.playlist_tracks(playlist_id, limit=100)
    tracks = results['items']

    while results['next']:
        results = sp.next(results)
        tracks.extend(results['items'])

    for item in tracks:
        track = item['track']
        if track:
            tracks_data.append({
                'Track Name': track.get('name'),
                'Artist': track['artists'][0].get('name') if track.get('artists') else None,
                'Album': track['album'].get('name') if track.get('album') else None,
                'Release Date': track['album'].get('release_date') if track.get('album') else None,
                'Duration (minutes)': track.get('duration_ms') / 60000 if track.get('duration_ms') else None,
                'Popularity': track.get('popularity'),
                'Track ID': track.get('id')
            })
    return tracks_data


# Example playlist
playlist_id = '6Mf614QiAuop5ud9x5beBS'  # All Time Pop Hits
tracks_data = get_playlist_tracks(sp, playlist_id)

df_playlist = pd.DataFrame(tracks_data)
print(f"\nCollected {len(df_playlist)} tracks from playlist!")


# Step 5: Data Cleaning & Transformation
df_playlist.drop_duplicates(subset='Track ID', inplace=True)
df_playlist.fillna({'Popularity': 0, 'Duration (minutes)': 0}, inplace=True)

# Convert Release Date to datetime
df_playlist['Release Date'] = pd.to_datetime(df_playlist['Release Date'], errors='coerce')

print("\nCleaned Playlist Data:")
print(df_playlist.head())


# Step 6: Load Data into SQL
# Setup SQLAlchemy engine (MySQL example)
engine = sqlalchemy.create_engine("mysql+pymysql://username:password@localhost/spotify_db")
df_playlist.to_sql('spotify_tracks', con=engine, if_exists='replace', index=False)
print("\nData loaded to SQL successfully!")


# Step 7: Analytical SQL Queries
query_top_artists = """
SELECT Artist, COUNT(`Track ID`) AS Total_Tracks, AVG(Popularity) AS Avg_Popularity
FROM spotify_tracks
GROUP BY Artist
ORDER BY Avg_Popularity DESC
LIMIT 10;
"""
top_artists = pd.read_sql(query_top_artists, engine)
print("\nTop Artists by Popularity:")
print(top_artists)


# Step 8: Visualizations
plt.figure(figsize=(10,6))
sns.barplot(data=top_artists, x='Artist', y='Avg_Popularity', palette='viridis')
plt.xticks(rotation=45)
plt.title('Top 10 Artists by Popularity')
plt.ylabel('Average Popularity')
plt.show()


plt.figure(figsize=(10,6))
sns.histplot(df_playlist['Duration (minutes)'], bins=30, kde=True, color='skyblue')
plt.title('Distribution of Track Duration')
plt.xlabel('Duration (minutes)')
plt.show()


# Step 9: Save to CSV
df_playlist.to_csv('spotify_playlist_data.csv', index=False)
df_track.to_csv('spotify_single_track.csv', index=False)
print("\nData saved to CSV files successfully!")
