from googleapiclient.discovery import build
import pandas as pd
import time

# ========== 1. 基本設定 ==========
API_KEY = "AIzaSyASKHCyoSeuJK3VFgmoS3DXZuaM47lWl-k"
youtube = build("youtube", "v3", developerKey=API_KEY)

# ========== 2. 抓大量熱門影片，蒐集候選頻道ID ==========
candidate_channels = set()
next_page_token = None
pages_to_fetch = 10  # 抓10頁，每頁50筆，共500筆熱門影片

for _ in range(pages_to_fetch):
    search_response = youtube.search().list(
        part="snippet",
        type="video",
        maxResults=50,
        order="viewCount",  # 熱門影片
        q="新聞",           # 可以改成其他分類
        pageToken=next_page_token
    ).execute()

    for item in search_response["items"]:
        candidate_channels.add(item["snippet"]["channelId"])

    next_page_token = search_response.get("nextPageToken")
    if not next_page_token:
        break

    time.sleep(0.1)  # 避免 API 限流

print(f"找到 {len(candidate_channels)} 個候選頻道")

# ========== 3. 找第一個影片數在100~1000的頻道 ==========
target_channel_id = None
for ch_id in candidate_channels:
    channel_data = youtube.channels().list(
        part="snippet,statistics,contentDetails",
        id=ch_id
    ).execute()["items"][0]

    video_count = int(channel_data["statistics"]["videoCount"])
    if 100 <= video_count <= 1000:
        target_channel_id = ch_id
        target_channel_name = channel_data["snippet"]["title"]
        uploads_playlist_id = channel_data["contentDetails"]["relatedPlaylists"]["uploads"]
        print(f"✅ 找到符合條件頻道：{target_channel_name}，影片數：{video_count}")
        break

if target_channel_id is None:
    print("⚠️ 沒有找到影片數在100~1000的頻道")
    exit()

# ========== 4. 抓取該頻道所有影片 ==========
videos = []
next_page_token = None
while True:
    playlist_response = youtube.playlistItems().list(
        part="snippet,contentDetails",
        playlistId=uploads_playlist_id,
        maxResults=50,
        pageToken=next_page_token
    ).execute()

    for item in playlist_response["items"]:
        videos.append({
            "channelName": target_channel_name,
            "channelId": target_channel_id,
            "videoId": item["contentDetails"]["videoId"],
            "title": item["snippet"]["title"],
            "uploadTime": item["contentDetails"]["videoPublishedAt"],
            "description": item["snippet"].get("description", "")
        })

    next_page_token = playlist_response.get("nextPageToken")
    if not next_page_token:
        break

    time.sleep(0.1)

# ========== 5. 整理成 DataFrame ==========
df = pd.DataFrame(videos)
print("✅ DataFrame 建立完成！")
print("Shape:", df.shape)
print("Columns:", df.columns.tolist())
print(df.head())

# ========== 6. 存成 CSV ==========
df.to_csv("first_filtered_channel_videos.csv", index=False, encoding="utf-8-sig")
print("✅ 已儲存成 first_filtered_channel_videos.csv")
