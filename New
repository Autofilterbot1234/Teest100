import os
import sys
import re
import requests
import json
import uuid
import math
from flask import Flask, render_template_string, request, redirect, url_for, Response, jsonify, flash
from pymongo import MongoClient
from bson.objectid import ObjectId
from dotenv import load_dotenv
from datetime import datetime

# --- কনফিগারেশন লোড ---
load_dotenv()

app = Flask(__name__)
app.secret_key = os.urandom(24)

# --- ভেরিয়েবলসমূহ ---
MONGO_URI = os.getenv("MONGO_URI")
TMDB_API_KEY = os.getenv("TMDB_API_KEY")
BOT_TOKEN = os.getenv("BOT_TOKEN")
BOT_USERNAME = os.getenv("BOT_USERNAME")
PUBLIC_CHANNEL_ID = os.getenv("PUBLIC_CHANNEL_ID")
SOURCE_CHANNEL_ID = os.getenv("SOURCE_CHANNEL_ID")
WEBSITE_URL = os.getenv("WEBSITE_URL")
TELEGRAM_API_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"

# এডমিন ক্রেডেনশিয়াল (এনভায়রনমেন্ট ভেরিয়েবল না থাকলে ডিফল্ট 'admin')
ADMIN_USER = os.getenv("ADMIN_USERNAME", "admin")
ADMIN_PASS = os.getenv("ADMIN_PASSWORD", "admin")

# --- ডেটাবেস কানেকশন ---
try:
    client = MongoClient(MONGO_URI)
    db = client["moviezone_db"]
    movies = db["movies"]
    settings = db["settings"]
    print("✅ MongoDB Connected Successfully!")
except Exception as e:
    print(f"❌ MongoDB Connection Error: {e}")
    sys.exit(1)

# === Helper Functions ===

def clean_filename(filename):
    name = os.path.splitext(filename)[0]
    name = re.sub(r'[._\-\[\]\(\)]', ' ', name)
    match = re.search(r'(\b(19|20)\d{2}\b|\b(?:480|720|1080|2160)[pP]\b|S\d+E\d+|Season)', name, re.IGNORECASE)
    if match:
        name = name[:match.start()]
    junk_words = r'\b(hindi|dual|audio|dubbed|sub|esub|web-dl|bluray|rip|x264|hevc)\b'
    name = re.sub(junk_words, '', name, flags=re.IGNORECASE)
    return name.strip()

def get_file_quality(filename):
    filename = filename.lower()
    if "4k" in filename or "2160p" in filename: return "4K UHD"
    if "1080p" in filename: return "1080p FHD"
    if "720p" in filename: return "720p HD"
    if "480p" in filename: return "480p SD"
    return "HD"

def get_tmdb_details(title, content_type="movie"):
    if not TMDB_API_KEY: return {"title": title}
    tmdb_type = "tv" if content_type == "series" else "movie"
    try:
        search_url = f"https://api.themoviedb.org/3/search/{tmdb_type}?api_key={TMDB_API_KEY}&query={requests.utils.quote(title)}"
        data = requests.get(search_url, timeout=5).json()
        if data.get("results"):
            res = data["results"][0]
            poster = f"https://image.tmdb.org/t/p/w500{res['poster_path']}" if res.get('poster_path') else None
            backdrop = f"https://image.tmdb.org/t/p/w1280{res['backdrop_path']}" if res.get('backdrop_path') else None
            return {
                "tmdb_id": res.get("id"),
                "title": res.get("name") if tmdb_type == "tv" else res.get("title"),
                "overview": res.get("overview"),
                "poster": poster,
                "backdrop": backdrop,
                "release_date": res.get("first_air_date") if tmdb_type == "tv" else res.get("release_date"),
                "vote_average": res.get("vote_average")
            }
    except Exception as e:
        print(f"TMDB Error: {e}")
    return {"title": title}

def escape_markdown(text):
    if not text: return ""
    chars = r'_*[]()~`>#+-=|{}.!'
    return re.sub(f'([{re.escape(chars)}])', r'\\\1', text)

def check_auth():
    auth = request.authorization
    if not auth or not (auth.username == ADMIN_USER and auth.password == ADMIN_PASS):
        return False
    return True

# === Context Processor ===
@app.context_processor
def inject_globals():
    ad_codes = settings.find_one() or {}
    return dict(
        ad_settings=ad_codes,
        BOT_USERNAME=BOT_USERNAME,
        site_name="MovieZone"
    )

# === TELEGRAM WEBHOOK ===
@app.route(f'/webhook/{BOT_TOKEN}', methods=['POST'])
def telegram_webhook():
    update = request.get_json()
    if not update: return jsonify({'status': 'ignored'})

    if 'channel_post' in update:
        msg = update['channel_post']
        chat_id = str(msg.get('chat', {}).get('id'))
        if SOURCE_CHANNEL_ID and chat_id != str(SOURCE_CHANNEL_ID):
            return jsonify({'status': 'wrong_channel'})

        file_id = None
        file_name = "Unknown"
        file_size_mb = 0
        file_type = "document"

        if 'video' in msg:
            video = msg['video']
            file_id = video['file_id']
            file_name = video.get('file_name', msg.get('caption', 'Unknown Video'))
            file_size_mb = video.get('file_size', 0) / (1024 * 1024)
            file_type = "video"
        elif 'document' in msg:
            doc = msg['document']
            file_id = doc['file_id']
            file_name = doc.get('file_name', 'Unknown Document')
            file_size_mb = doc.get('file_size', 0) / (1024 * 1024)
            file_type = "document"

        if not file_id: return jsonify({'status': 'no_file'})

        raw_caption = msg.get('caption')
        raw_input = raw_caption if raw_caption else file_name
        search_title = clean_filename(raw_input)
        
        content_type = "movie"
        if re.search(r'(S\d+|Season)', file_name, re.IGNORECASE) or re.search(r'(S\d+|Season)', str(raw_caption), re.IGNORECASE):
            content_type = "series"

        tmdb_data = get_tmdb_details(search_title, content_type)
        final_title = tmdb_data.get('title', search_title)
        quality = get_file_quality(file_name)
        unique_code = str(uuid.uuid4())[:8]

        language = "Unknown"
        if re.search(r'\b(hindi|dual)\b', raw_input, re.IGNORECASE):
            language = "Hindi / Dual"
        elif re.search(r'\b(bangla|bengali)\b', raw_input, re.IGNORECASE):
            language = "Bengali"
        else:
            language = "English"

        file_obj = {
            "file_id": file_id,
            "unique_code": unique_code,
            "filename": file_name,
            "quality": quality,
            "size": f"{file_size_mb:.2f} MB",
            "file_type": file_type,
            "added_at": datetime.utcnow()
        }

        existing_movie = movies.find_one({"title": final_title})

        if existing_movie:
            movies.update_one(
                {"_id": existing_movie['_id']},
                {"$push": {"files": file_obj}, "$set": {"updated_at": datetime.utcnow()}}
            )
            movie_id = existing_movie['_id']
        else:
            new_movie = {
                "title": final_title,
                "overview": tmdb_data.get('overview'),
                "poster": tmdb_data.get('poster'),
                "backdrop": tmdb_data.get('backdrop'),
                "release_date": tmdb_data.get('release_date'),
                "vote_average": tmdb_data.get('vote_average'),
                "language": language, 
                "type": content_type,
                "files": [file_obj],
                "created_at": datetime.utcnow(),
                "updated_at": datetime.utcnow()
            }
            res = movies.insert_one(new_movie)
            movie_id = res.inserted_id

        if movie_id and WEBSITE_URL:
            dl_link = f"{WEBSITE_URL.rstrip('/')}/movie/{str(movie_id)}"
            edit_payload = {
                'chat_id': chat_id,
                'message_id': msg['message_id'],
                'reply_markup': json.dumps({
                    "inline_keyboard": [[
                        {"text": "▶️ Download from Website", "url": dl_link}
                    ]]
                })
            }
            try: requests.post(f"{TELEGRAM_API_URL}/editMessageReplyMarkup", json=edit_payload)
            except: pass

        return jsonify({'status': 'success'})

    elif 'message' in update:
        msg = update['message']
        chat_id = msg.get('chat', {}).get('id')
        text = msg.get('text', '')

        if text.startswith('/start'):
            parts = text.split()
            if len(parts) > 1:
                code = parts[1]
                movie = movies.find_one({"files.unique_code": code})
                if movie:
                    target_file = next((f for f in movie['files'] if f['unique_code'] == code), None)
                    if target_file:
                        caption = f"🎬 *{escape_markdown(movie['title'])}*\n"
                        caption += f"🔊 Audio: {movie.get('language', 'N/A')}\n"
                        caption += f"💿 Quality: {target_file['quality']}\n"
                        caption += f"📦 Size: {target_file['size']}\n\n"
                        caption += f"✅ *Downloaded from {escape_markdown(WEBSITE_URL)}*"
                        
                        payload = {'chat_id': chat_id, 'caption': caption, 'parse_mode': 'Markdown'}
                        method = 'sendVideo' if target_file['file_type'] == 'video' else 'sendDocument'
                        if target_file['file_type'] == 'video': payload['video'] = target_file['file_id']
                        else: payload['document'] = target_file['file_id']
                        requests.post(f"{TELEGRAM_API_URL}/{method}", json=payload)
                    else:
                        requests.post(f"{TELEGRAM_API_URL}/sendMessage", json={'chat_id': chat_id, 'text': "❌ File expired."})
                else:
                    requests.post(f"{TELEGRAM_API_URL}/sendMessage", json={'chat_id': chat_id, 'text': "❌ Invalid Link."})
            else:
                requests.post(f"{TELEGRAM_API_URL}/sendMessage", json={'chat_id': chat_id, 'text': "👋 Welcome! Use the website to download movies."})

    return jsonify({'status': 'ok'})

# ================================
#        FRONTEND TEMPLATES
# ================================

index_template = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>{{ site_name }} - Home</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap');
        :root { --primary: #E50914; --dark: #0f0f0f; --card-bg: #1a1a1a; --text: #fff; }
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Poppins', sans-serif; -webkit-tap-highlight-color: transparent; }
        body { background-color: var(--dark); color: var(--text); padding-bottom: 70px; }
        a { text-decoration: none; color: inherit; }
        
        .navbar { display: flex; justify-content: space-between; align-items: center; padding: 12px 15px; background: rgba(0,0,0,0.8); backdrop-filter: blur(10px); position: sticky; top: 0; z-index: 100; border-bottom: 1px solid #222; }
        .logo { font-size: 20px; font-weight: 700; color: var(--primary); text-transform: uppercase; }
        .search-box { position: relative; }
        .search-box input { background: #222; border: 1px solid #333; padding: 8px 15px; padding-right: 35px; border-radius: 20px; color: #fff; outline: none; width: 140px; font-size: 13px; transition: width 0.3s; }
        .search-box input:focus { width: 180px; border-color: var(--primary); }
        .search-box i { position: absolute; right: 12px; top: 50%; transform: translateY(-50%); color: #777; font-size: 12px; pointer-events: none; }

        .hero { height: 280px; background-size: cover; background-position: center; position: relative; display: flex; align-items: flex-end; margin-bottom: 20px; }
        .hero::after { content: ''; position: absolute; inset: 0; background: linear-gradient(to top, var(--dark) 0%, transparent 100%); }
        .hero-content { position: relative; z-index: 2; padding: 20px; width: 100%; }
        .hero-title { font-size: 1.8rem; line-height: 1.2; text-shadow: 0 2px 4px rgba(0,0,0,0.8); margin-bottom: 5px; }
        .btn-play { background: var(--primary); border: none; padding: 8px 20px; border-radius: 4px; color: #fff; font-weight: 600; font-size: 0.9rem; display: inline-flex; align-items: center; gap: 8px; margin-top: 10px; }

        .section { padding: 0 15px; }
        .section-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 15px; }
        .section-title { font-size: 1.1rem; border-left: 3px solid var(--primary); padding-left: 10px; font-weight: 600; }

        .grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
        @media (min-width: 600px) { .grid { grid-template-columns: repeat(4, 1fr); gap: 15px; } }
        @media (min-width: 900px) { .grid { grid-template-columns: repeat(6, 1fr); gap: 20px; } }

        .card { position: relative; background: var(--card-bg); border-radius: 6px; overflow: hidden; aspect-ratio: 2/3; }
        .card-img { width: 100%; height: 100%; object-fit: cover; }
        .card-overlay { position: absolute; inset: 0; background: linear-gradient(to top, rgba(0,0,0,0.9) 0%, transparent 60%); display: flex; flex-direction: column; justify-content: flex-end; padding: 8px; }
        .card-title { font-size: 0.8rem; font-weight: 500; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; line-height: 1.2; }
        .card-meta { font-size: 0.7rem; color: #ccc; margin-top: 2px; display: flex; justify-content: space-between; }
        
        .rating-badge { position: absolute; top: 6px; left: 6px; background: rgba(0,0,0,0.7); color: #ffb400; padding: 2px 5px; border-radius: 3px; font-size: 0.65rem; font-weight: bold; backdrop-filter: blur(4px); }
        .lang-badge { position: absolute; top: 6px; right: 6px; background: var(--primary); color: #fff; padding: 2px 6px; border-radius: 3px; font-size: 0.6rem; font-weight: 600; text-transform: uppercase; box-shadow: 0 2px 4px rgba(0,0,0,0.3); }

        .bottom-nav { position: fixed; bottom: 0; width: 100%; background: #161616; display: flex; justify-content: space-around; padding: 10px 0; border-top: 1px solid #252525; z-index: 99; backdrop-filter: blur(10px); }
        .nav-item { display: flex; flex-direction: column; align-items: center; color: #777; font-size: 10px; transition: 0.2s; }
        .nav-item i { font-size: 18px; margin-bottom: 4px; }
        .nav-item.active { color: var(--primary); }
        
        .ad-container { margin: 15px 0; text-align: center; min-height: 50px; background: #111; border-radius: 4px; overflow: hidden; }
    </style>
</head>
<body>

<nav class="navbar">
    <a href="/" class="logo">{{ site_name }}</a>
    <form action="/" method="GET" class="search-box">
        <input type="text" name="q" placeholder="Search..." value="{{ query }}">
        <i class="fas fa-search"></i>
    </form>
</nav>

{% if not query and featured %}
<div class="hero" style="background-image: url('{{ featured.backdrop or featured.poster }}');">
    <div class="hero-content">
        <h1 class="hero-title">{{ featured.title }}</h1>
        <div style="font-size: 0.8rem; color: #ddd; margin-bottom: 5px;">
            <i class="fas fa-star" style="color: #ffb400;"></i> {{ featured.vote_average }} | {{ featured.language }}
        </div>
        <a href="{{ url_for('movie_detail', movie_id=featured._id) }}" class="btn-play">
            <i class="fas fa-play"></i> Watch Now
        </a>
    </div>
</div>
{% endif %}

<main class="section">
    {% if ad_settings.banner_ad %}<div class="ad-container">{{ ad_settings.banner_ad|safe }}</div>{% endif %}

    <div class="section-header">
        <h2 class="section-title">{{ query and 'Search Results' or 'Latest Uploads' }}</h2>
    </div>

    <div class="grid">
        {% for movie in movies %}
        <a href="{{ url_for('movie_detail', movie_id=movie._id) }}" class="card">
            <span class="rating-badge">{{ movie.vote_average }}</span>
            {% if movie.language %}<span class="lang-badge">{{ movie.language }}</span>{% endif %}
            <img src="{{ movie.poster or 'https://via.placeholder.com/300x450?text=No+Poster' }}" class="card-img" loading="lazy">
            <div class="card-overlay">
                <h3 class="card-title">{{ movie.title }}</h3>
                <div class="card-meta">
                    <span>{{ (movie.release_date or 'N/A')[:4] }}</span>
                    <span>{{ movie.type|capitalize }}</span>
                </div>
            </div>
        </a>
        {% endfor %}
    </div>
    
    <div style="height: 20px;"></div>
</main>

<nav class="bottom-nav">
    <a href="/" class="nav-item {{ 'active' if not request.args.get('type') else '' }}"><i class="fas fa-home"></i>Home</a>
    <a href="/movies" class="nav-item {{ 'active' if request.args.get('type') == 'movie' else '' }}"><i class="fas fa-film"></i>Movies</a>
    <a href="/series" class="nav-item {{ 'active' if request.args.get('type') == 'series' else '' }}"><i class="fas fa-tv"></i>Series</a>
</nav>

{% if ad_settings.popunder %}{{ ad_settings.popunder|safe }}{% endif %}

</body>
</html>
"""

detail_template = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>{{ movie.title }} - Download</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600;700&display=swap');
        :root { --primary: #E50914; --dark: #0f0f0f; --bg-sec: #1a1a1a; --text: #eee; }
        body { background-color: var(--dark); color: var(--text); font-family: 'Poppins', sans-serif; padding-bottom: 30px; }
        .container { max-width: 900px; margin: 0 auto; padding: 15px; }
        
        .backdrop { height: 250px; position: relative; overflow: hidden; margin-bottom: -80px; }
        .backdrop img { width: 100%; height: 100%; object-fit: cover; opacity: 0.6; mask-image: linear-gradient(to bottom, black 50%, transparent 100%); }
        .back-btn { position: absolute; top: 15px; left: 15px; background: rgba(0,0,0,0.6); color: #fff; width: 35px; height: 35px; display: flex; align-items: center; justify-content: center; border-radius: 50%; z-index: 10; font-size: 14px; }
        
        .movie-info { position: relative; display: flex; flex-direction: column; align-items: center; text-align: center; gap: 15px; z-index: 5; }
        .poster-box { width: 140px; border-radius: 8px; box-shadow: 0 5px 15px rgba(0,0,0,0.5); overflow: hidden; border: 2px solid #333; }
        .poster-box img { width: 100%; display: block; }
        
        h1 { font-size: 1.6rem; margin-bottom: 5px; line-height: 1.2; }
        .meta-tags { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; margin-bottom: 15px; font-size: 0.8rem; color: #bbb; }
        .tag { background: #333; padding: 3px 8px; border-radius: 4px; }
        .overview { font-size: 0.9rem; line-height: 1.6; color: #ccc; margin-bottom: 25px; text-align: justify; }
        
        .file-section { background: var(--bg-sec); border-radius: 8px; padding: 15px; border: 1px solid #2a2a2a; }
        .section-head { font-size: 1rem; margin-bottom: 15px; display: flex; align-items: center; gap: 10px; color: var(--primary); font-weight: 600; border-bottom: 1px solid #333; padding-bottom: 10px; }
        
        /* Updated Button Styles */
        .file-item { display: flex; flex-direction: column; align-items: center; background: #252525; padding: 15px; border-radius: 8px; margin-bottom: 12px; text-align: center; }
        .file-details h4 { font-size: 1rem; margin-bottom: 4px; color: #fff; }
        .file-details span { font-size: 0.8rem; color: #999; }
        
        .btn-dl { 
            background: #0088cc; 
            color: white; 
            width: 100%; /* Full width */
            padding: 10px; 
            margin-top: 10px; 
            border-radius: 6px; 
            text-decoration: none; 
            font-weight: 600; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
            gap: 8px;
            font-size: 0.95rem;
            transition: 0.3s;
        }
        .btn-dl:hover { background: #0077b5; transform: translateY(-2px); }
        
        @media (min-width: 600px) {
            .movie-info { flex-direction: row; text-align: left; align-items: flex-end; padding: 0 20px; }
            .overview { text-align: left; padding: 0 20px; }
            .backdrop { height: 350px; margin-bottom: -100px; }
            .poster-box { width: 180px; }
        }
    </style>
</head>
<body>

<a href="/" class="back-btn"><i class="fas fa-arrow-left"></i></a>

<div class="backdrop">
    <img src="{{ movie.backdrop or movie.poster }}" alt="">
</div>

<div class="container">
    <div class="movie-info">
        <div class="poster-box">
            <img src="{{ movie.poster }}" alt="Poster">
        </div>
        <div style="padding-bottom: 10px;">
            <h1>{{ movie.title }}</h1>
            <div class="meta-tags">
                <span class="tag"><i class="fas fa-star" style="color:#ffb400"></i> {{ movie.vote_average }}</span>
                <span class="tag">{{ (movie.release_date or 'N/A')[:4] }}</span>
                <span class="tag" style="background: var(--primary); color: #fff;">{{ movie.language or 'Eng' }}</span>
                <span class="tag">{{ movie.type|upper }}</span>
            </div>
        </div>
    </div>
    
    <div style="height: 20px;"></div>
    <p class="overview">{{ movie.overview }}</p>

    {% if ad_settings.banner_ad %}<div style="margin: 20px 0; text-align:center;">{{ ad_settings.banner_ad|safe }}</div>{% endif %}

    <div class="file-section">
        <div class="section-head"><i class="fas fa-download"></i> Download Links</div>
        {% if movie.files %}
            {% for file in movie.files|reverse %}
            <div class="file-item">
                <div class="file-details">
                    <h4>{{ file.quality }}</h4>
                    <span>Size: {{ file.size }} • Format: {{ file.file_type|upper }}</span>
                </div>
                <a href="https://t.me/{{ BOT_USERNAME }}?start={{ file.unique_code }}" class="btn-dl">
                    <i class="fab fa-telegram-plane"></i> Get File
                </a>
            </div>
            {% endfor %}
        {% else %}
            <p style="text-align: center; color: #666; font-size: 0.9rem;">No files added yet.</p>
        {% endif %}
    </div>
    
    <div style="text-align: center; margin-top: 20px; font-size: 0.8rem; color: #555;">
        &copy; {{ site_name }} 2025
    </div>
</div>

{% if ad_settings.popunder %}{{ ad_settings.popunder|safe }}{% endif %}

</body>
</html>
"""

# ================================
#        ADMIN PANEL TEMPLATES (FIXED)
# ================================

admin_base = """
<!DOCTYPE html>
<html lang="en" data-bs-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Panel - MovieZone</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #0f1012; }
        .sidebar { height: 100vh; position: fixed; top: 0; left: 0; width: 240px; background: #191b1f; padding-top: 20px; border-right: 1px solid #2a2d31; }
        .sidebar a { padding: 12px 25px; display: block; color: #aaa; text-decoration: none; transition: 0.3s; font-weight: 500; }
        .sidebar a:hover, .sidebar a.active { color: #fff; background: #E50914; border-radius: 0 25px 25px 0; }
        .sidebar .brand { font-size: 22px; font-weight: bold; color: #E50914; text-align: center; margin-bottom: 30px; }
        .main-content { margin-left: 240px; padding: 30px; }
        .card { background: #1f2226; border: 1px solid #2a2d31; }
        .form-control { background: #131517; border-color: #333; color: #fff; }
        .form-control:focus { background: #131517; color: #fff; border-color: #E50914; box-shadow: none; }
        .poster-preview { width: 100%; border-radius: 8px; max-width: 200px; }
        @media (max-width: 768px) {
            .sidebar { width: 60px; }
            .sidebar a span, .sidebar .brand span { display: none; }
            .sidebar a { padding: 15px; text-align: center; border-radius: 0; }
            .main-content { margin-left: 60px; padding: 15px; }
        }
    </style>
</head>
<body>

<div class="sidebar">
    <div class="brand"><i class="fas fa-play-circle"></i> <span>Admin</span></div>
    <a href="/admin" class="{{ 'active' if active == 'dashboard' else '' }}"><i class="fas fa-th-large"></i> <span>Movies</span></a>
    <a href="/admin/settings" class="{{ 'active' if active == 'settings' else '' }}"><i class="fas fa-cogs"></i> <span>Settings</span></a>
    <a href="/" target="_blank"><i class="fas fa-external-link-alt"></i> <span>View Site</span></a>
</div>

<div class="main-content">
    <!-- CONTENT_GOES_HERE -->
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
<script>
    function searchTMDB() {
        const query = document.getElementById('tmdbQuery').value;
        const resultDiv = document.getElementById('tmdbResults');
        if(!query) return;
        
        resultDiv.innerHTML = '<div class="text-info">Searching...</div>';
        
        fetch('/admin/api/tmdb?q=' + encodeURIComponent(query))
        .then(r => r.json())
        .then(data => {
            if(data.error) {
                resultDiv.innerHTML = '<div class="text-danger">'+data.error+'</div>';
                return;
            }
            let html = '<div class="list-group mt-2">';
            data.results.forEach(item => {
                let title = item.title || item.name;
                let date = item.release_date || item.first_air_date || 'N/A';
                let itemStr = JSON.stringify(item).replace(/'/g, "&#39;");
                
                html += `<button type="button" class="list-group-item list-group-item-action d-flex align-items-center gap-3" onclick='fillForm(${itemStr})'>
                    <img src="https://image.tmdb.org/t/p/w92${item.poster_path}" style="width:40px; border-radius:4px;">
                    <div>
                        <div class="fw-bold">${title}</div>
                        <small class="text-muted">${date.substring(0,4)}</small>
                    </div>
                </button>`;
            });
            html += '</div>';
            resultDiv.innerHTML = html;
        });
    }

    function fillForm(data) {
        document.querySelector('input[name="title"]').value = data.title || data.name;
        document.querySelector('textarea[name="overview"]').value = data.overview;
        document.querySelector('input[name="poster"]').value = 'https://image.tmdb.org/t/p/w500' + data.poster_path;
        document.querySelector('input[name="backdrop"]').value = 'https://image.tmdb.org/t/p/w1280' + data.backdrop_path;
        document.querySelector('input[name="release_date"]').value = data.release_date || data.first_air_date;
        document.querySelector('input[name="vote_average"]').value = data.vote_average;
        document.querySelector('.poster-preview').src = 'https://image.tmdb.org/t/p/w500' + data.poster_path;
        document.getElementById('tmdbResults').innerHTML = '<div class="text-success">Data Applied! Click Update to Save.</div>';
    }
</script>
</body>
</html>
"""

admin_dashboard = """
<div class="d-flex justify-content-between align-items-center mb-4">
    <h2>Manage Movies</h2>
    <form class="d-flex" method="GET">
        <input class="form-control me-2" type="search" name="q" placeholder="Search movies..." value="{{ q }}">
        <button class="btn btn-outline-light" type="submit">Search</button>
    </form>
</div>

<div class="row">
    {% for movie in movies %}
    <div class="col-md-6 col-lg-4 col-xl-3 mb-4">
        <div class="card h-100">
            <div class="row g-0 h-100">
                <div class="col-4">
                    <img src="{{ movie.poster or 'https://via.placeholder.com/150' }}" class="img-fluid rounded-start h-100" style="object-fit:cover;" alt="...">
                </div>
                <div class="col-8">
                    <div class="card-body p-2 d-flex flex-column h-100">
                        <h6 class="card-title mb-1 text-truncate">{{ movie.title }}</h6>
                        <p class="card-text small text-muted mb-1">{{ (movie.release_date or '')[:4] }} • {{ movie.language }}</p>
                        <div class="mt-auto d-flex gap-2">
                            <a href="/admin/movie/edit/{{ movie._id }}" class="btn btn-sm btn-primary flex-grow-1">Edit</a>
                            <a href="/admin/movie/delete/{{ movie._id }}" onclick="return confirm('Are you sure?')" class="btn btn-sm btn-danger"><i class="fas fa-trash"></i></a>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    {% endfor %}
</div>

<div class="d-flex justify-content-center mt-4">
    {% if page > 1 %}
    <a href="?page={{ page-1 }}&q={{ q }}" class="btn btn-outline-secondary me-2">Previous</a>
    {% endif %}
    <span class="align-self-center mx-2">Page {{ page }}</span>
    <a href="?page={{ page+1 }}&q={{ q }}" class="btn btn-outline-secondary ms-2">Next</a>
</div>
"""

admin_edit = """
<div class="container-fluid">
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h3>Edit Movie: <span class="text-primary">{{ movie.title }}</span></h3>
        <a href="/admin" class="btn btn-secondary btn-sm">Back</a>
    </div>

    <div class="row">
        <!-- TMDB Search Column -->
        <div class="col-md-4 mb-4">
            <div class="card p-3">
                <h5>Fetch Data from TMDB</h5>
                <div class="input-group mb-3">
                    <input type="text" id="tmdbQuery" class="form-control" placeholder="Enter Movie Name..." value="{{ movie.title }}">
                    <button class="btn btn-warning" type="button" onclick="searchTMDB()">Search</button>
                </div>
                <div id="tmdbResults" style="max-height: 400px; overflow-y: auto;"></div>
            </div>
            
            <div class="card p-3 mt-3 text-center">
                <label class="form-label">Current Poster</label><br>
                <img src="{{ movie.poster }}" class="poster-preview">
            </div>
        </div>

        <!-- Edit Form Column -->
        <div class="col-md-8">
            <div class="card p-4">
                <form method="POST">
                    <div class="row">
                        <div class="col-md-8 mb-3">
                            <label class="form-label">Movie Title</label>
                            <input type="text" name="title" class="form-control" value="{{ movie.title }}" required>
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Language (Badge)</label>
                            <input type="text" name="language" class="form-control" value="{{ movie.language }}" placeholder="e.g. Hindi, Dual Audio">
                        </div>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Overview / Story</label>
                        <textarea name="overview" class="form-control" rows="4">{{ movie.overview }}</textarea>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Poster URL</label>
                            <input type="text" name="poster" class="form-control" value="{{ movie.poster }}">
                        </div>
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Backdrop URL</label>
                            <input type="text" name="backdrop" class="form-control" value="{{ movie.backdrop }}">
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Release Date</label>
                            <input type="text" name="release_date" class="form-control" value="{{ movie.release_date }}">
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">TMDB Rating</label>
                            <input type="text" name="vote_average" class="form-control" value="{{ movie.vote_average }}">
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Type</label>
                            <select name="type" class="form-control">
                                <option value="movie" {{ 'selected' if movie.type == 'movie' else '' }}>Movie</option>
                                <option value="series" {{ 'selected' if movie.type == 'series' else '' }}>Series</option>
                            </select>
                        </div>
                    </div>

                    <button type="submit" class="btn btn-primary btn-lg w-100">Update Movie Details</button>
                </form>
            </div>
        </div>
    </div>
</div>
"""

admin_settings = """
<div class="container" style="max-width: 800px;">
    <h3 class="mb-4">Website Settings</h3>
    <div class="card p-4">
        <form method="POST">
            <div class="mb-4">
                <label class="form-label fw-bold text-warning">Banner Ad Code (HTML)</label>
                <div class="form-text mb-2">This code appears on the Homepage and Download Page.</div>
                <textarea name="banner_ad" class="form-control" rows="5" style="font-family: monospace;">{{ settings.banner_ad }}</textarea>
            </div>
            
            <div class="mb-4">
                <label class="form-label fw-bold text-warning">Popunder / Scripts (HTML/JS)</label>
                <div class="form-text mb-2">Use this for Popunders, Google Analytics, or other hidden scripts.</div>
                <textarea name="popunder" class="form-control" rows="5" style="font-family: monospace;">{{ settings.popunder }}</textarea>
            </div>
            
            <button type="submit" class="btn btn-success"><i class="fas fa-save"></i> Save Settings</button>
        </form>
    </div>
</div>
"""

# ================================
#        FLASK ROUTES
# ================================

@app.route('/')
def home():
    query = request.args.get('q', '').strip()
    filter_type = request.args.get('type')
    
    db_query = {}
    if query:
        db_query["title"] = {"$regex": query, "$options": "i"}
    if filter_type:
        db_query["type"] = filter_type

    movie_list = list(movies.find(db_query).sort([('updated_at', -1), ('_id', -1)]).limit(24))
    
    featured = None
    if not query and not filter_type and movie_list:
        featured = movie_list[0] 

    return render_template_string(index_template, movies=movie_list, query=query, featured=featured)

@app.route('/movies')
def view_movies():
    return redirect(url_for('home', type='movie'))

@app.route('/series')
def view_series():
    return redirect(url_for('home', type='series'))

@app.route('/movie/<movie_id>')
def movie_detail(movie_id):
    try:
        movie = movies.find_one({"_id": ObjectId(movie_id)})
        if not movie: return "Movie Not Found", 404
        return render_template_string(detail_template, movie=movie)
    except:
        return "Invalid ID", 400

# ================================
#        ADMIN ROUTES (FIXED)
# ================================

@app.route('/admin')
def admin_home():
    if not check_auth():
        return Response('Login Required', 401, {'WWW-Authenticate': 'Basic realm="Login Required"'})
    
    page = int(request.args.get('page', 1))
    q = request.args.get('q', '')
    per_page = 20
    
    filter_q = {}
    if q: filter_q['title'] = {'$regex': q, '$options': 'i'}
    
    movie_list = list(movies.find(filter_q).sort('_id', -1).skip((page-1)*per_page).limit(per_page))
    
    # Fix: Manually combine base + dashboard strings
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_dashboard)
    return render_template_string(full_html, movies=movie_list, page=page, q=q, active='dashboard')

@app.route('/admin/movie/edit/<movie_id>', methods=['GET', 'POST'])
def admin_edit_movie(movie_id):
    if not check_auth(): return Response('Login Required', 401)
    
    if request.method == 'POST':
        update_data = {
            "title": request.form.get("title"),
            "language": request.form.get("language"),
            "overview": request.form.get("overview"),
            "poster": request.form.get("poster"),
            "backdrop": request.form.get("backdrop"),
            "release_date": request.form.get("release_date"),
            "vote_average": request.form.get("vote_average"),
            "type": request.form.get("type"),
            "updated_at": datetime.utcnow()
        }
        movies.update_one({"_id": ObjectId(movie_id)}, {"$set": update_data})
        return redirect(url_for('admin_home'))
        
    movie = movies.find_one({"_id": ObjectId(movie_id)})
    
    # Fix: Manually combine base + edit strings
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_edit)
    return render_template_string(full_html, movie=movie, active='dashboard')

@app.route('/admin/movie/delete/<movie_id>')
def admin_delete_movie(movie_id):
    if not check_auth(): return Response('Login Required', 401)
    movies.delete_one({"_id": ObjectId(movie_id)})
    return redirect(url_for('admin_home'))

@app.route('/admin/settings', methods=['GET', 'POST'])
def admin_settings_page():
    if not check_auth(): return Response('Login Required', 401)
    
    if request.method == 'POST':
        settings.update_one({}, {"$set": {
            "banner_ad": request.form.get("banner_ad"),
            "popunder": request.form.get("popunder")
        }}, upsert=True)
        return redirect(url_for('admin_settings_page'))
    
    curr_settings = settings.find_one() or {}
    
    # Fix: Manually combine base + settings strings
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_settings)
    return render_template_string(full_html, settings=curr_settings, active='settings')

# API for Admin Panel (JS Fetch)
@app.route('/admin/api/tmdb')
def api_tmdb_search():
    if not check_auth(): return jsonify({'error': 'Unauthorized'}), 401
    query = request.args.get('q')
    if not query or not TMDB_API_KEY: return jsonify({'error': 'No query or API key'})
    
    url = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&query={requests.utils.quote(query)}"
    try:
        data = requests.get(url).json()
        return jsonify(data)
    except:
        return jsonify({'error': 'TMDB Request Failed'})

if __name__ == '__main__':
    # Webhook Auto-Set
    if WEBSITE_URL and BOT_TOKEN:
        hook_url = f"{WEBSITE_URL.rstrip('/')}/webhook/{BOT_TOKEN}"
        try: requests.get(f"https://api.telegram.org/bot{BOT_TOKEN}/setWebhook?url={hook_url}")
        except: pass

    port = int(os.environ.get("PORT", 5000))
    app.run(host='0.0.0.0', port=port, debug=True)
