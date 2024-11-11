
# YouTube Playlist & Videos Catalog in Port

This guide will help you set up an automated process to catalog YouTube playlist and video data into Port. Using Port's GitHub action, you’ll fetch YouTube data and ingest it into Port for easy tracking and visualization.

## Prerequisites

1. [Create a Port account](https://app.getport.io) and set up API credentials.
2. [Obtain a YouTube Data API Key](https://console.cloud.google.com/apis/credentials).
3. [Set up GitHub secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) in your repository for:
   - `YOUTUBE_API_KEY`: Your YouTube API key.
   - `CLIENT_ID`: Your Port client ID.
   - `CLIENT_SECRET`: Your Port client secret.

## Step 1: Model Data in Port

Define two blueprints in Port: `youtube_playlist` for playlists and `youtube_video` for individual videos.

### Playlist Blueprint (`youtube_playlist`)

- **Properties**:
  - `title` (string): The title of the playlist.
  - `link` (string): YouTube URL of the playlist.
  - `description` (string): Description of the playlist.
  - `publishedAt` (string): Publish date of the playlist.
  - `channelId` (string): ID of the YouTube channel.
  - `channelTitle` (string): Title of the YouTube channel.
  - `thumbnails` (object): Thumbnail images (default, medium, high, standard).
  - `localized` (object): Localized title and description.

---

   <details>
     <summary>Configuration mapping for playlist blueprint (click to expand)</summary>

```json showLineNumbers
{
    "identifier": "youtube_playlist",
    "title": "YouTube Playlist",
    "description": "Blueprint for YouTube Playlist",
    "icon": "",
    "schema": {
        "properties": {
            "id": { "type": "string" },
            "title": { "type": "string" },
            "link": { "type": "string" },
            "description": { "type": "string" },
            "publishedAt": { "type": "string" },
            "channelId": { "type": "string" },
            "channelTitle": { "type": "string" },
            "thumbnails": {
                "type": "object",
                "properties": {
                    "default": { "type": "string" },
                    "medium": { "type": "string" },
                    "high": { "type": "string" },
                    "standard": { "type": "string" }
                }
            },
            "localized": {
                "type": "object",
                "properties": {
                    "title": {"type": "string"},
                    "description": {"type": "string"}
                }
            }
        },
        "required": ["id", "title", "description", "publishedAt", "channelId", "channelTitle"]
    }
}
```
   </details>

---

### Video Blueprint (`youtube_video`)

- **Properties**:
  - `title` (string): Title of the video.
  - `link` (string): YouTube URL of the video.
  - `duration` (string): Duration of the video.
  - `description` (string): Description of the video.
  - `publishedAt` (string): Publish date of the video.
  - `position` (integer): Position in the playlist.
  - `thumbnails` (object): Thumbnail images (default, medium, high, standard, maxres).
  - `videoOwnerChannelTitle` (string): Title of the owner channel.
  - `videoOwnerChannelId` (string): ID of the owner channel.
- **Relationships**:
  - `playlist`: Links to the `youtube_playlist` entity.

---

   <details>
     <summary>Configuration mapping for video blueprint (click to expand)</summary>
     
```json showLineNumbers
{
    "identifier": "youtube_video",
    "title": "YouTube Video",
    "description": "Blueprint for YouTube Video",
    "icon": "",
    "schema": {
        "properties": {
            "id": { "type": "string" },
            "title": { "type": "string" },
            "link": { "type": "string" },
            "duration": { "type": "string" },
            "description": { "type": "string" },
            "publishedAt": { "type": "string" },
            "position": { "type": "integer" },
            "thumbnails": {
                "type": "object",
                "properties": {
                    "default": { "type": "string" },
                    "medium": { "type": "string" },
                    "high": { "type": "string" },
                    "standard": { "type": "string" },
                    "maxres": { "type": "string" }
                }
            },
            "videoOwnerChannelTitle": { "type": "string" },
            "videoOwnerChannelId": { "type": "string" }
        },
        "required": ["id", "title", "description", "publishedAt", "duration", "videoLink"]
    },
    "relations": {
        "playlist": {
            "title": "Playlist",
            "many": false,
            "target": "youtube_playlist",
            "required": true
        }
    }
}

```
   </details>

---

## Step 2: GitHub Workflow for Data Ingestion

The following GitHub workflow automates fetching data from YouTube and updating Port with the data.

### GitHub Workflow (`.github/workflows/youtube_port_workflow.yml`)

```yaml showLineNumbers
name: Update YouTube Playlist and Video Entities in Port

on:
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * *'  # Runs daily at midnight

jobs:
  update_port_entities:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run YouTube Data Fetch and Prepare Port Entities
        env:
          YOUTUBE_API_KEY: ${{ secrets.YOUTUBE_API_KEY }}
          CLIENT_ID: ${{ secrets.CLIENT_ID }}
          CLIENT_SECRET: ${{ secrets.CLIENT_SECRET }}
        run: python fetch_youtube_data.py

      - name: Bulk Create/Update YouTube Playlist and Video Entities in Port
        uses: port-labs/port-github-action@v1
        with:
          clientId: ${{ secrets.CLIENT_ID }}
          clientSecret: ${{ secrets.CLIENT_SECRET }}
          baseUrl: https://api.getport.io
          operation: BULK_UPSERT
          entities: ${{ toJson(fromJson(file('port_entities.json'))) }}
```

---

### Fetch YouTube Data

The `fetch_youtube_data.py` script retrieves YouTube data and prepares it in the required JSON format.

```python
import requests
import json
from googleapiclient.discovery import build
import os
import logging
from dotenv import load_dotenv

# Configure logging
logging.basicConfig(level=logging.INFO)
load_dotenv()

YOUTUBE_API_KEY = os.getenv("YOUTUBE_API_KEY")
PLAYLIST_ID = "YOUR_PLAYLIST_ID"

def fetch_youtube_playlist_data(api_key, playlist_id):
    youtube = build("youtube", "v3", developerKey=api_key)
    videos = []
    # Additional logic to fetch video details
    # ...
    return videos

def fetch_youtube_playlist_info(api_key, playlist_id):
    youtube = build("youtube", "v3", developerKey=api_key)
    request = youtube.playlists().list(
        part="snippet",
        id=playlist_id
    )
    # Additional logic to fetch playlist details
    # ...
    return {
        "identifier": playlist_id,
        "blueprint": "youtube_playlist",
        # Additional playlist props
        # ...
    }

def main():
    playlist_data = fetch_youtube_playlist_info(YOUTUBE_API_KEY, PLAYLIST_ID)
    videos_data = fetch_youtube_playlist_data(YOUTUBE_API_KEY, PLAYLIST_ID)
    # Combine Playlist and Video data for BULK_UPSERT
    all_data = [playlist_data] + videos_data
    with open("port_entities.json", "w") as f:
        json.dump(all_data, f, indent=4)
    logging.info("Fetched YouTube data and saved to port_entities.json")

if __name__ == "__main__":
    main()
```
---

   <details>
     <summary>Fetch YouTube Data complete implementation (click to expand)</summary>
     
```python showLineNumbers
import requests
import json
from googleapiclient.discovery import build
import os
import logging
from dotenv import load_dotenv

# Configure logging
logging.basicConfig(level=logging.INFO)

load_dotenv()  # Load environment variables from .env file

# Client credentials
YOUTUBE_API_KEY = os.getenv("YOUTUBE_API_KEY")

# YouTube playlist details
PLAYLIST_ID = "PL5ErBr2d3QJH0kbwTQ7HSuzvBb4zIWzhy"


def fetch_youtube_playlist_data(api_key, playlist_id):
    youtube = build("youtube", "v3", developerKey=api_key)
    videos = []
    next_page_token = None

    while True:
        playlist_request = youtube.playlistItems().list(
            part="snippet,contentDetails", playlistId=playlist_id, maxResults=10, pageToken=next_page_token
        )
        playlist_response = playlist_request.execute()
        
        for item in playlist_response["items"]:
            video_id = item["contentDetails"]["videoId"]
            
            # Fetch additional video details including duration
            video_request = youtube.videos().list(
                part="contentDetails",
                id=video_id
            )
            video_response = video_request.execute()
            duration = video_response["items"][0]["contentDetails"]["duration"]

            title = item["snippet"]["title"]
            description = item["snippet"]["description"]
            publishedAt = item["snippet"]["publishedAt"]
            position = item["snippet"].get("position", None)
            thumbnails = item["snippet"]["thumbnails"]
            videoOwnerChannelTitle = item["snippet"].get("videoOwnerChannelTitle", "")
            videoOwnerChannelId = item["snippet"].get("videoOwnerChannelId", "")
            video_link = f"https://www.youtube.com/watch?v={video_id}"

            videos.append({
                "identifier": video_id,
                "blueprint": "youtube_video",
                "properties": {
                    "title": title,
                    "duration": duration,
                    "link": video_link,
                    "description": description,
                    "publishedAt": publishedAt,
                    "position": position,
                    "thumbnails": {
                        "default": thumbnails["default"]["url"],
                        "medium": thumbnails["medium"]["url"],
                        "high": thumbnails["high"]["url"],
                        "standard": thumbnails.get("standard", {}).get("url")
                    },
                    "videoOwnerChannelTitle": videoOwnerChannelTitle,
                    "videoOwnerChannelId": videoOwnerChannelId
                },
                "relations": {
                    "playlist": PLAYLIST_ID
                }
            })

        next_page_token = playlist_response.get("nextPageToken")
        if not next_page_token:
            break

    return videos


def fetch_youtube_playlist_info(api_key, playlist_id):
    youtube = build("youtube", "v3", developerKey=api_key)
    request = youtube.playlists().list(
        part="snippet",
        id=playlist_id
    )
    response = request.execute()
    item = response["items"][0]
    title = item["snippet"]["title"]
    description = item["snippet"]["description"]
    published_at = item["snippet"]["publishedAt"]
    channel_id = item["snippet"]["channelId"]
    channel_title = item["snippet"]["channelTitle"]
    thumbnails = item["snippet"]["thumbnails"]
    playlist_link = f"https://www.youtube.com/playlist?list={playlist_id}"
    localized_title = item["snippet"]["localized"]["title"]
    localized_description = item["snippet"]["localized"]["description"]

    return {
        "identifier": playlist_id,
        "blueprint": "youtube_playlist",
        "properties": {
            "title": title,
            "link": playlist_link,
            "description": description,
            "publishedAt": published_at,
            "channelId": channel_id,
            "channelTitle": channel_title,
            "thumbnails": {
                "default": thumbnails["default"]["url"],
                "medium": thumbnails["medium"]["url"],
                "high": thumbnails["high"]["url"],
                "standard": thumbnails.get("standard", {}).get("url")
            },
            "localized": {
                "title": localized_title,
                "description": localized_description
            }
        }
    }


def main():
    playlist_data = fetch_youtube_playlist_info(YOUTUBE_API_KEY, PLAYLIST_ID)
    videos_data = fetch_youtube_playlist_data(YOUTUBE_API_KEY, PLAYLIST_ID)
    
    # Combine Playlist and Video data for BULK_UPSERT
    all_data = [playlist_data] + videos_data
    with open("port_entities.json", "w") as f:
        json.dump(all_data, f, indent=4)
    logging.info("Fetched YouTube data and saved to port_entities.json")


if __name__ == "__main__":
    main()

```
   </details>

---

### Explanation of Workflow Steps

1. **Check out the code**: Retrieves the repository code.
2. **Set up Python**: Configures Python 3.9 environment.
3. **Install dependencies**: Installs required packages from `requirements.txt`.
4. **Run YouTube Data Fetch**: Runs `fetch_youtube_data.py` to retrieve YouTube data and prepare it for Port ingestion.
5. **Bulk Create/Update Entities**: Uses Port’s GitHub action to upsert the playlist and video data.

## Step 3: Visualizing Data in Port

Once data is ingested, you can visualize it in Port by setting up:
1. A dashboard for tracking playlist-level metrics like the number of videos, publish dates, and durations.
2. Video-level insights, such as view count, position in playlist, and thumbnail displays.

