# server.py
from flask import Flask, request, jsonify
import yt_dlp
import os

app = Flask(__name__)

YDL_OPTS = {
    "format": "best[ext=mp4]/best",
    "quiet": True,
    "no_warnings": True,
    "writesubtitles": True,
    "writeautomaticsub": True,
    "subtitlesformat": "vtt",
    "skip_download": True
}

@app.route("/video/stream")
def stream_video():
    video_id = request.args.get("video_id")
    if not video_id:
        return jsonify({"error": "video_id missing"}), 400

    url = f"https://www.youtube.com/watch?v={video_id}"

    try:
        with yt_dlp.YoutubeDL(YDL_OPTS) as ydl:
            info = ydl.extract_info(url, download=False)

            # MP4 STREAM
            stream_url = None
            for f in info.get("formats", []):
                if f.get("ext") == "mp4" and f.get("acodec") != "none":
                    stream_url = f.get("url")
                    break

            # TITLOVI
            captions = []
            subs = info.get("subtitles") or info.get("automatic_captions")

            if subs:
                for lang, tracks in subs.items():
                    for t in tracks:
                        if t.get("ext") == "vtt":
                            captions.append({
                                "language": lang,
                                "url": t.get("url")
                            })
                            break
                    break  # samo jedan jezik za početak

            return jsonify({
                "video_id": video_id,
                "title": info.get("title"),
                "stream_url": stream_url,
                "captions": captions
            })

    except Exception as e:
        return jsonify({"error": str(e)}), 500


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=10000)
    
