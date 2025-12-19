import os
import sys
import re
import requests
import json
import uuid
import math
from flask import Flask, render_template_string, request, redirect, url_for, Response, jsonify
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

# এডমিন ক্রেডেনশিয়াল
ADMIN_USER = os.getenv("ADMIN_USERNAME", "admin")
ADMIN_PASS = os.getenv("ADMIN_PASSWORD", "admin")

# --- ডেটাবেস কানেকশন ---
try:
    client = MongoClient(MONGO_URI)
    db = client["moviezone_db"]
    movies = db["movies"]
    settings = db["settings"]
    categories = db["categories"] # নতুন ক্যাটাগরি কালেকশন
    print("✅ MongoDB Connected Successfully!")
except Exception as e:
    print(f"❌ MongoDB Connection Error: {e}")
    sys.exit(1)

# === Helper Functions ===

def clean_filename(filename):
    """ ফাইলের নাম ক্লিন করে মেইন টাইটেল বের করে। """
    name = os.path.splitext(filename)[0]
    name = re.sub(r'[._\-\+\[\]\(\)]', ' ', name)
    stop_pattern = r'(\b(19|20)\d{2}\b|\bS\d+|\bSeason|\bEp?\s*\d+|\b480p|\b720p|\b1080p|\b2160p|\bHD|\bWeb-?dl|\bBluray|\bDual|\bHindi|\bBangla)'
    match = re.search(stop_pattern, name, re.IGNORECASE)
    if match:
        name = name[:match.start()]
    name = re.sub(r'\s+', ' ', name).strip()
    return name

def get_file_quality(filename):
    filename = filename.lower()
    if "4k" in filename or "2160p" in filename: return "4K UHD"
    if "1080p" in filename: return "1080p FHD"
    if "720p" in filename: return "720p HD"
    if "480p" in filename: return "480p SD"
    return "HD"

def detect_language(text):
    text = text.lower()
    detected = []
    if re.search(r'\b(multi|multi audio)\b', text): return "Multi Audio"
    if re.search(r'\b(dual|dual audio)\b', text): detected.append("Dual Audio")

    lang_map = {
        'Bengali': ['bengali', 'bangla', 'ben'],
        'Hindi': ['hindi', 'hin'],
        'English': ['english', 'eng'],
        'Tamil': ['tamil', 'tam'],
        'Telugu': ['telugu', 'tel'],
        'Korean': ['korean', 'kor'],
        'Japanese': ['japanese', 'jap']
    }
    for lang_name, keywords in lang_map.items():
        pattern = r'\b(' + '|'.join(keywords) + r')\b'
        if re.search(pattern, text):
            detected.append(lang_name)

    if not detected: return "English"
    return " + ".join(list(dict.fromkeys(detected)))

def get_episode_label(filename):
    label = ""
    season = ""
    match_s = re.search(r'\b(S|Season)\s*(\d+)', filename, re.IGNORECASE)
    if match_s: season = f"S{int(match_s.group(2)):02d}"

    match_range = re.search(r'E(\d+)\s*-\s*E?(\d+)', filename, re.IGNORECASE)
    if match_range:
        start, end = int(match_range.group(1)), int(match_range.group(2))
        episode_part = f"E{start:02d}-{end:02d}"
        return f"{season} {episode_part}" if season else episode_part

    match_se = re.search(r'\bS(\d+)\s*E(\d+)\b', filename, re.IGNORECASE)
    if match_se: return f"S{int(match_se.group(1)):02d} E{int(match_se.group(2)):02d}"
    
    match_ep = re.search(r'\b(Episode|Ep|E)\s*(\d+)\b', filename, re.IGNORECASE)
    if match_ep:
        ep_num = int(match_ep.group(2))
        if ep_num < 1900: return f"{season} Episode {ep_num}".strip()
    
    if season: return f"Season {int(match_s.group(2))}"
    return None

# --- TMDB FUNCTION ---
def get_tmdb_details(title, content_type="movie", year=None):
    if not TMDB_API_KEY: return {"title": title}
    tmdb_type = "tv" if content_type == "series" else "movie"
    try:
        query_str = requests.utils.quote(title)
        search_url = f"https://api.themoviedb.org/3/search/{tmdb_type}?api_key={TMDB_API_KEY}&query={query_str}"
        if year and tmdb_type == "movie":
            search_url += f"&year={year}"

        data = requests.get(search_url, timeout=5).json()
        if data.get("results"):
            res = data["results"][0]
            m_id = res.get("id")
            
            # --- Fetch Extra Details (Credits, Videos) ---
            details_url = f"https://api.themoviedb.org/3/{tmdb_type}/{m_id}?api_key={TMDB_API_KEY}&append_to_response=credits,videos"
            extra = requests.get(details_url, timeout=5).json()

            # Extract Trailer
            trailer_key = None
            if extra.get('videos', {}).get('results'):
                for vid in extra['videos']['results']:
                    if vid['type'] == 'Trailer' and vid['site'] == 'YouTube':
                        trailer_key = vid['key']
                        break
            
            # Extract Cast
            cast_list = []
            if extra.get('credits', {}).get('cast'):
                for actor in extra['credits']['cast'][:6]:
                    cast_list.append({
                        'name': actor['name'],
                        'img': f"https://image.tmdb.org/t/p/w185{actor['profile_path']}" if actor.get('profile_path') else None
                    })

            # Genres & Runtime
            genres = [g['name'] for g in extra.get('genres', [])]
            runtime = extra.get("runtime") or (extra.get("episode_run_time")[0] if extra.get("episode_run_time") else None)

            poster = f"https://image.tmdb.org/t/p/w500{res['poster_path']}" if res.get('poster_path') else None
            backdrop = f"https://image.tmdb.org/t/p/w1280{res['backdrop_path']}" if res.get('backdrop_path') else None

            return {
                "tmdb_id": res.get("id"),
                "title": res.get("name") if tmdb_type == "tv" else res.get("title"),
                "overview": res.get("overview"),
                "poster": poster,
                "backdrop": backdrop,
                "release_date": res.get("first_air_date") if tmdb_type == "tv" else res.get("release_date"),
                "vote_average": res.get("vote_average"),
                "genres": genres,        
                "runtime": runtime, 
                "trailer": trailer_key,  
                "cast": cast_list
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
    return dict(ad_settings=ad_codes, BOT_USERNAME=BOT_USERNAME, site_name="MovieZone")

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
        year_match = re.search(r'\b(19|20)\d{2}\b', raw_input)
        search_year = year_match.group(0) if year_match else None
        
        content_type = "movie"
        if re.search(r'(S\d+|Season|Episode|Ep\s*\d+|Combined|E\d+-E\d+)', file_name, re.IGNORECASE) or re.search(r'(S\d+|Season)', str(raw_caption), re.IGNORECASE):
            content_type = "series"

        tmdb_data = get_tmdb_details(search_title, content_type, search_year)
        final_title = tmdb_data.get('title', search_title)
        quality = get_file_quality(file_name)
        
        episode_label = get_episode_label(file_name)
        if content_type == "series" and not episode_label:
            clean_part = file_name.replace(search_title, "").replace(".", " ").strip()
            if len(clean_part) > 3:
                episode_label = clean_part[:25]

        language = detect_language(raw_input)
        unique_code = str(uuid.uuid4())[:8]

        file_obj = {
            "file_id": file_id,
            "unique_code": unique_code,
            "filename": file_name,
            "quality": quality,
            "episode_label": episode_label,
            "size": f"{file_size_mb:.2f} MB",
            "file_type": file_type,
            "added_at": datetime.utcnow()
        }

        existing_movie = movies.find_one({"title": final_title})
        movie_id = None
        should_notify = False

        # --- DUPLICATE CHECK LOGIC ---
        if existing_movie:
            is_duplicate = False
            for f in existing_movie.get('files', []):
                # একই ফাইল আইডি থাকলে ডুপ্লিকেট
                if f.get('file_id') == file_id:
                    is_duplicate = True
                    break
            
            # যদি ডুপ্লিকেট না হয়, তবেই অ্যাড করবে
            if not is_duplicate:
                movies.update_one(
                    {"_id": existing_movie['_id']},
                    {"$push": {"files": file_obj}, "$set": {"updated_at": datetime.utcnow()}}
                )
                movie_id = existing_movie['_id']
                # নোটিফিকেশন লজিক
                if content_type == "series" and episode_label:
                    # সিরিজের নতুন এপিসোড আসলে নোটিফাই করবে, যদি সেই এপিসোড আগে না থাকে
                    has_ep = False
                    for f in existing_movie.get('files', []):
                        if f.get('episode_label') == episode_label and f.get('quality') == quality and f != file_obj:
                            has_ep = True
                            break
                    should_notify = not has_ep
                else:
                    should_notify = False
        else:
            should_notify = True
            new_movie = {
                "title": final_title,
                "overview": tmdb_data.get('overview'),
                "poster": tmdb_data.get('poster'),
                "backdrop": tmdb_data.get('backdrop'),
                "release_date": tmdb_data.get('release_date'),
                "vote_average": tmdb_data.get('vote_average'),
                "genres": tmdb_data.get('genres'),
                "runtime": tmdb_data.get('runtime'),
                "trailer": tmdb_data.get('trailer'),
                "cast": tmdb_data.get('cast'),
                "language": language,
                "type": content_type,
                "category": "Uncategorized", # ডিফল্ট ক্যাটাগরি
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
                    "inline_keyboard": [[{"text": "▶️ Download from Website", "url": dl_link}]]
                })
            }
            try: requests.post(f"{TELEGRAM_API_URL}/editMessageReplyMarkup", json=edit_payload)
            except: pass

            if PUBLIC_CHANNEL_ID and should_notify:
                notify_caption = f"🎬 *{escape_markdown(final_title)}*\n"
                if episode_label: notify_caption += f"📌 {escape_markdown(episode_label)}\n"
                
                notify_caption += f"\n⭐ Rating: {tmdb_data.get('vote_average', 'N/A')}\n"
                notify_caption += f"📅 Year: {(tmdb_data.get('release_date') or 'N/A')[:4]}\n"
                notify_caption += f"🔊 Language: {language}\n"
                notify_caption += f"💿 Quality: {quality}\n"
                notify_caption += f"📦 Size: {file_size_mb:.2f} MB\n\n"
                notify_caption += f"🔗 *Download Now:* [Click Here]({dl_link})"

                notify_payload = {
                    'chat_id': PUBLIC_CHANNEL_ID,
                    'parse_mode': 'Markdown',
                    'reply_markup': json.dumps({"inline_keyboard": [[{"text": "📥 Download / Watch Online", "url": dl_link}]]})
                }

                if tmdb_data.get('poster'):
                    notify_payload['photo'] = tmdb_data.get('poster')
                    notify_payload['caption'] = notify_caption
                    try: requests.post(f"{TELEGRAM_API_URL}/sendPhoto", json=notify_payload)
                    except: pass
                else:
                    notify_payload['text'] = notify_caption
                    try: requests.post(f"{TELEGRAM_API_URL}/sendMessage", json=notify_payload)
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
                        if target_file.get('episode_label'):
                            caption += f"📌 {escape_markdown(target_file['episode_label'])}\n"
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
        @import url('https://fonts.googleapis.com/css2?family=Hind+Siliguri:wght@400;500;600;700&display=swap'); /* For Bangla Text */
        
        :root { --primary: #E50914; --dark: #0f1012; --card-bg: #1a1a1a; --text: #fff; --red-btn: #cc0000; }
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Poppins', sans-serif; -webkit-tap-highlight-color: transparent; }
        body { background-color: var(--dark); color: var(--text); padding-bottom: 70px; }
        a { text-decoration: none; color: inherit; }
        
        /* NAVBAR */
        .navbar { display: flex; justify-content: space-between; align-items: center; padding: 12px 15px; background: #161616; border-bottom: 1px solid #222; }
        .logo { font-size: 22px; font-weight: 800; color: var(--primary); text-transform: uppercase; letter-spacing: 1px; }
        .nav-icons { color: #fff; font-size: 18px; }

        /* CATEGORY GRID (RED BUTTONS) */
        .category-container {
            padding: 15px 10px;
            background: #121212;
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            gap: 8px;
        }

        .cat-btn {
            background: var(--red-btn);
            color: white;
            padding: 8px 12px;
            border-radius: 6px;
            font-size: 13px;
            font-weight: 700;
            text-transform: uppercase;
            display: inline-flex;
            align-items: center;
            gap: 6px;
            border: 1px solid #990000;
            box-shadow: 0 3px 0 #800000; /* 3D Effect */
            transition: transform 0.1s, box-shadow 0.1s;
            white-space: nowrap;
        }

        .cat-btn:active {
            transform: translateY(3px);
            box-shadow: none;
        }
        
        .cat-btn.active {
            background: #ffcc00;
            color: #000;
            border-color: #cc9900;
            box-shadow: 0 3px 0 #997700;
        }

        .cat-btn i, .cat-btn span.emoji { font-size: 14px; }

        /* BIG SEARCH BAR (Like Image) */
        .search-wrapper {
            padding: 5px 15px 20px 15px;
            background: #121212;
            display: flex;
            justify-content: center;
        }
        .big-search-box {
            width: 100%;
            max-width: 600px;
            display: flex;
            background: #1e252b;
            border: 2px solid #00c3ff;
            border-radius: 8px;
            overflow: hidden;
            box-shadow: 0 0 10px rgba(0, 195, 255, 0.2);
        }
        .big-search-box input {
            flex: 1;
            background: transparent;
            border: none;
            padding: 12px 15px;
            color: #fff;
            font-family: 'Hind Siliguri', sans-serif;
            font-size: 16px;
            outline: none;
        }
        .big-search-box button {
            background: #00c3ff;
            border: none;
            width: 55px;
            cursor: pointer;
            color: #fff;
            font-size: 20px;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        /* HERO & CONTENT */
        .hero { height: 280px; background-size: cover; background-position: center; position: relative; display: flex; align-items: flex-end; margin-bottom: 20px; margin-top: 10px;}
        .hero::after { content: ''; position: absolute; inset: 0; background: linear-gradient(to top, var(--dark) 0%, transparent 100%); }
        .hero-content { position: relative; z-index: 2; padding: 20px; width: 100%; }
        .hero-title { font-size: 1.8rem; line-height: 1.2; text-shadow: 0 2px 4px rgba(0,0,0,0.8); margin-bottom: 5px; }
        .btn-play { background: var(--primary); border: none; padding: 8px 20px; border-radius: 4px; color: #fff; font-weight: 600; font-size: 0.9rem; display: inline-flex; align-items: center; gap: 8px; margin-top: 10px; }

        .section { padding: 0 15px; }
        .section-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 15px; border-left: 4px solid var(--primary); padding-left: 10px; }
        .section-title { font-size: 1.2rem; font-weight: 700; text-transform: uppercase; }

        .grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; }
        @media (min-width: 600px) { .grid { grid-template-columns: repeat(3, 1fr); gap: 15px; } }
        @media (min-width: 900px) { .grid { grid-template-columns: repeat(5, 1fr); gap: 20px; } }

        .card { position: relative; background: var(--card-bg); border-radius: 6px; overflow: hidden; aspect-ratio: 2/3; transition: transform 0.2s; }
        .card:hover { transform: scale(1.03); z-index: 10; }
        .card-img { width: 100%; height: 100%; object-fit: cover; }
        .card-overlay { position: absolute; inset: 0; background: linear-gradient(to top, rgba(0,0,0,0.95) 0%, transparent 60%); display: flex; flex-direction: column; justify-content: flex-end; padding: 10px; }
        .card-title { font-size: 0.85rem; font-weight: 500; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; line-height: 1.2; }
        .card-meta { font-size: 0.75rem; color: #ccc; margin-top: 2px; display: flex; justify-content: space-between; }
        
        .rating-badge { position: absolute; top: 6px; left: 6px; background: rgba(0,0,0,0.7); color: #ffb400; padding: 2px 5px; border-radius: 3px; font-size: 0.65rem; font-weight: bold; backdrop-filter: blur(4px); }
        .lang-badge { position: absolute; top: 6px; right: 6px; background: var(--primary); color: #fff; padding: 2px 6px; border-radius: 3px; font-size: 0.6rem; font-weight: 600; text-transform: uppercase; box-shadow: 0 2px 4px rgba(0,0,0,0.3); }

        .pagination { display: flex; justify-content: center; gap: 10px; margin: 30px 0; }
        .page-btn { padding: 8px 16px; background: #222; border-radius: 4px; color: #fff; font-size: 0.9rem; border: 1px solid #333; transition: 0.2s; }
        .page-btn:hover { background: #333; }

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
    <div class="nav-icons">
        <i class="fas fa-bell"></i>
    </div>
</nav>

<!-- RED CATEGORY BUTTONS GRID -->
<div class="category-container">
    <a href="/" class="cat-btn {{ 'active' if not selected_cat and not request.args.get('type') else '' }}">
        <span class="emoji">🏠</span> Home
    </a>
    
    <!-- Static Categories like Image -->
    <a href="/movies" class="cat-btn {{ 'active' if request.args.get('type') == 'movie' else '' }}">
        <i class="fas fa-film"></i> All Movies
    </a>
    <a href="/series" class="cat-btn {{ 'active' if request.args.get('type') == 'series' else '' }}">
        <i class="fas fa-tv"></i> All Web Series
    </a>

    <!-- Dynamic Categories from DB -->
    {% for cat in categories %}
    <a href="/?cat={{ cat.name }}" class="cat-btn {{ 'active' if selected_cat == cat.name else '' }}">
        <!-- Simple logic to assign icons based on name -->
        {% if 'Bangla' in cat.name %}<span class="emoji">🇧🇩</span>
        {% elif 'Hindi' in cat.name %}<span class="emoji">🇮🇳</span>
        {% elif 'English' in cat.name %}<span class="emoji">🇺🇸</span>
        {% elif 'Korean' in cat.name %}<span class="emoji">🇰🇷</span>
        {% elif 'Dual' in cat.name %}<span class="emoji">🎧</span>
        {% elif 'Action' in cat.name %}<span class="emoji">⚔️</span>
        {% elif 'Horror' in cat.name %}<span class="emoji">👻</span>
        {% elif 'Love' in cat.name or 'Romance' in cat.name %}<span class="emoji">💘</span>
        {% elif 'Thriller' in cat.name %}<span class="emoji">🧩</span>
        {% else %}<i class="fas fa-tag"></i>{% endif %}
        {{ cat.name }}
    </a>
    {% endfor %}
</div>

<!-- BIG SEARCH BAR -->
<div class="search-wrapper">
    <form action="/" method="GET" class="big-search-box">
        <input type="text" name="q" placeholder="সার্চ করে খুঁজে নিন পছন্দের মুভি..." value="{{ query }}">
        <button type="submit"><i class="fas fa-search"></i></button>
    </form>
</div>

{% if not query and not selected_cat and featured %}
<div class="hero" style="background-image: url('{{ featured.backdrop or featured.poster }}');">
    <div class="hero-content">
        <h1 class="hero-title">{{ featured.title }}</h1>
        <div style="font-size: 0.8rem; color: #ddd; margin-bottom: 5px;">
            <i class="fas fa-star" style="color: #ffb400;"></i> {{ featured.vote_average }} | {{ featured.category or 'Movie' }}
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
        <h2 class="section-title">{{ selected_cat if selected_cat else (query and 'Search Results' or 'Latest Uploads') }}</h2>
    </div>

    <div class="grid">
        {% for movie in movies %}
        <a href="{{ url_for('movie_detail', movie_id=movie._id) }}" class="card">
            <span class="rating-badge">{{ movie.vote_average }}</span>
            {% if movie.language %}<span class="lang-badge">{{ movie.language[:3] }}</span>{% endif %}
            <img src="{{ movie.poster or 'https://via.placeholder.com/300x450?text=No+Poster' }}" class="card-img" loading="lazy">
            <div class="card-overlay">
                <h3 class="card-title">{{ movie.title }}</h3>
                <div class="card-meta">
                    <span>{{ (movie.release_date or 'N/A')[:4] }}</span>
                    <span>{{ movie.category or 'Movie' }}</span>
                </div>
            </div>
        </a>
        {% endfor %}
    </div>

    <!-- PAGINATION -->
    <div class="pagination">
        {% if page > 1 %}
        <a href="/?page={{ page-1 }}&cat={{ selected_cat or '' }}&q={{ query or '' }}" class="page-btn">Previous</a>
        {% endif %}
        {% if has_next %}
        <a href="/?page={{ page+1 }}&cat={{ selected_cat or '' }}&q={{ query or '' }}" class="page-btn">Next</a>
        {% endif %}
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
# --- DETAIL TEMPLATE (Fixed Cast & Trailer) ---
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
        .meta-tags { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; margin-bottom: 8px; font-size: 0.8rem; color: #bbb; }
        .tag { background: #333; padding: 3px 8px; border-radius: 4px; }
        .overview { font-size: 0.9rem; line-height: 1.6; color: #ccc; margin-bottom: 25px; text-align: justify; }
        
        .file-section { background: var(--bg-sec); border-radius: 8px; padding: 15px; border: 1px solid #2a2a2a; }
        .section-head { font-size: 1rem; margin-bottom: 15px; display: flex; align-items: center; gap: 10px; color: var(--primary); font-weight: 600; border-bottom: 1px solid #333; padding-bottom: 10px; }
        
        .file-item { display: flex; flex-direction: column; align-items: center; background: #252525; padding: 15px; border-radius: 8px; margin-bottom: 12px; text-align: center; }
        .file-details h4 { font-size: 1rem; margin-bottom: 4px; color: #fff; }
        .file-details span { font-size: 0.8rem; color: #999; }
        
        .btn-dl { 
            background: #0088cc; 
            color: white; 
            width: 100%;
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

        .badge-q { padding: 3px 8px; border-radius: 4px; font-size: 0.7rem; font-weight: bold; }
        .q-4k { background: #d63384; color: #fff; box-shadow: 0 0 10px rgba(214, 51, 132, 0.5); }
        .q-1080p { background: #6f42c1; color: #fff; }
        .q-720p { background: #0d6efd; color: #fff; }
        .q-480p { background: #198754; color: #fff; }

        @media (min-width: 600px) {
            .movie-info { flex-direction: row; text-align: left; align-items: flex-end; padding: 0 20px; }
            .meta-tags { justify-content: flex-start; }
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
                {% if movie.runtime %}<span class="tag"><i class="far fa-clock"></i> {{ movie.runtime }} min</span>{% endif %}
                <span class="tag" style="background: var(--primary); color: #fff;">{{ movie.language or 'Eng' }}</span>
                <span class="tag">{{ movie.category or 'Movie' }}</span>
            </div>
            <div style="margin-top:5px; font-size: 0.85rem; color: #aaa;">
                {% if movie.genres %}
                    {{ movie.genres|join(', ') }}
                {% endif %}
            </div>
        </div>
    </div>
    
    <div style="height: 20px;"></div>
    <p class="overview">{{ movie.overview }}</p>

    <!-- Trailer Section -->
    {% if movie.trailer %}
    <div style="margin-bottom: 25px; padding: 0 5px;">
        <div class="section-head"><i class="fab fa-youtube"></i> Watch Trailer</div>
        <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; border-radius: 8px; border: 1px solid #333;">
            <iframe style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" 
                    src="https://www.youtube.com/embed/{{ movie.trailer }}" allowfullscreen></iframe>
        </div>
    </div>
    {% endif %}

    <!-- Cast Section -->
    {% if movie.cast %}
    <div style="margin-bottom: 25px; padding: 0 5px;">
        <div class="section-head"><i class="fas fa-users"></i> Top Cast</div>
        <div style="display: flex; gap: 15px; overflow-x: auto; padding-bottom: 10px; scrollbar-width: none;">
            {% for actor in movie.cast %}
            <div style="min-width: 90px; text-align: center;">
                <img src="{{ actor.img or 'https://via.placeholder.com/90x90?text=No+Img' }}" 
                     style="width: 80px; height: 80px; border-radius: 50%; object-fit: cover; border: 2px solid #333; margin-bottom: 5px;">
                <div style="font-size: 0.75rem; color: #ccc; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;">{{ actor.name }}</div>
            </div>
            {% endfor %}
        </div>
    </div>
    {% endif %}

    {% if ad_settings.banner_ad %}<div style="margin: 20px 0; text-align:center;">{{ ad_settings.banner_ad|safe }}</div>{% endif %}

    <div style="display: flex; gap: 10px; margin-bottom: 20px;">
        <a href="whatsapp://send?text=Download {{ movie.title }} - {{ request.url }}" class="btn-dl" style="background: #25D366; flex: 1; margin:0;">
            <i class="fab fa-whatsapp"></i> Share
        </a>
        <a href="https://www.facebook.com/sharer/sharer.php?u={{ request.url }}" target="_blank" class="btn-dl" style="background: #1877F2; flex: 1; margin:0;">
            <i class="fab fa-facebook-f"></i> Share
        </a>
        <button onclick="navigator.clipboard.writeText(window.location.href); alert('Link Copied!')" class="btn-dl" style="background: #333; flex: 1; margin:0;">
            <i class="fas fa-link"></i> Copy
        </button>
    </div>

    <div class="file-section">
        <div class="section-head"><i class="fas fa-download"></i> Download Links</div>
        {% if movie.files %}
            {% for file in movie.files|reverse %}
            <div class="file-item">
                <div class="file-details">
                    {% if file.episode_label %}
                        <h4 style="color: #ffb400; font-weight: 700;">{{ file.episode_label }}</h4>
                        {% set q_class = 'q-480p' %}
                        {% if '1080p' in file.quality %} {% set q_class = 'q-1080p' %}
                        {% elif '720p' in file.quality %} {% set q_class = 'q-720p' %}
                        {% elif '4K' in file.quality %} {% set q_class = 'q-4k' %}
                        {% endif %}
                        <span class="badge-q {{ q_class }}">{{ file.quality }}</span>
                    {% else %}
                        <h4>{{ file.quality }}</h4>
                    {% endif %}
                    
                    <div style="font-size: 0.75rem; color: #888; margin-top: 3px;">
                        Size: {{ file.size }} • Format: {{ file.file_type|upper }}
                    </div>
                    <div style="font-size: 0.65rem; color: #555; margin-top: 2px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 250px;">
                        {{ file.filename }}
                    </div>
                </div>
                
                <a href="https://t.me/{{ BOT_USERNAME }}?start={{ file.unique_code }}" class="btn-dl">
                    <i class="fab fa-telegram-plane"></i> 
                    {% if file.episode_label %}
                        Watch {{ file.episode_label }}
                    {% else %}
                        Get File
                    {% endif %}
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
#        ADMIN PANEL TEMPLATES
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
    <a href="/admin/categories" class="{{ 'active' if active == 'categories' else '' }}"><i class="fas fa-tags"></i> <span>Categories</span></a>
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
                        <span class="badge bg-danger mb-1">{{ movie.category }}</span>
                        <p class="card-text small text-muted mb-1">{{ (movie.release_date or '')[:4] }}</p>
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

# --- NEW CATEGORY MANAGEMENT TEMPLATE ---
admin_categories = """
<div class="container" style="max-width: 600px;">
    <h3 class="mb-4">Manage Categories</h3>
    
    <div class="card p-3 mb-4">
        <form method="POST" class="d-flex gap-2">
            <input type="text" name="new_category" class="form-control" placeholder="New Category Name (e.g. Bollywood)" required>
            <button class="btn btn-success" type="submit">Add</button>
        </form>
    </div>

    <div class="list-group">
        {% for cat in categories %}
        <div class="list-group-item d-flex justify-content-between align-items-center bg-dark text-white border-secondary">
            <span>{{ cat.name }}</span>
            <a href="/admin/categories/delete/{{ cat._id }}" class="btn btn-sm btn-danger"><i class="fas fa-trash"></i></a>
        </div>
        {% endfor %}
    </div>
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
                            <label class="form-label">Category</label>
                            <select name="category" class="form-select">
                                <option value="Uncategorized">Select Category...</option>
                                {% for cat in categories %}
                                <option value="{{ cat.name }}" {{ 'selected' if movie.category == cat.name else '' }}>{{ cat.name }}</option>
                                {% endfor %}
                            </select>
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
                            <label class="form-label">Rating</label>
                            <input type="text" name="vote_average" class="form-control" value="{{ movie.vote_average }}">
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Language</label>
                            <input type="text" name="language" class="form-control" value="{{ movie.language }}">
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
    page = int(request.args.get('page', 1))
    per_page = 16 # পেজ প্রতি ১৬টি মুভি দেখাবে
    query = request.args.get('q', '').strip()
    cat_filter = request.args.get('cat', '').strip()
    type_filter = request.args.get('type', '').strip()
    
    db_query = {}
    if query:
        db_query["title"] = {"$regex": query, "$options": "i"}
    if cat_filter:
        db_query["category"] = cat_filter
    if type_filter:
        db_query["type"] = type_filter

    # মোট মুভির সংখ্যা (পেজিনেশনের জন্য)
    total_movies = movies.count_documents(db_query)
    
    # স্কিপ এবং লিমিট ব্যবহার করে নির্দিষ্ট পেজের মুভি আনা
    movie_list = list(movies.find(db_query).sort([('updated_at', -1), ('_id', -1)]).skip((page-1)*per_page).limit(per_page))
    
    # সব ক্যাটাগরি আনা (ন্যাভিগেশনের জন্য)
    cat_list = list(categories.find())
    
    featured = None
    if not query and not cat_filter and not type_filter and movie_list:
        featured = movies.find_one(sort=[('vote_average', -1)]) # সর্বোচ্চ রেটিং মুভি ফিচারড হবে

    has_next = (page * per_page) < total_movies

    return render_template_string(index_template, movies=movie_list, categories=cat_list, selected_cat=cat_filter, query=query, featured=featured, page=page, has_next=has_next)

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
#        ADMIN ROUTES
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
    
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_dashboard)
    return render_template_string(full_html, movies=movie_list, page=page, q=q, active='dashboard')

# নতুন ক্যাটাগরি রাউট
@app.route('/admin/categories', methods=['GET', 'POST'])
def admin_cats():
    if not check_auth(): return Response('Login Required', 401)
    
    if request.method == 'POST':
        new_cat = request.form.get('new_category').strip()
        if new_cat:
            categories.insert_one({"name": new_cat})
        return redirect(url_for('admin_cats'))
    
    cat_list = list(categories.find())
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_categories)
    return render_template_string(full_html, categories=cat_list, active='categories')

@app.route('/admin/categories/delete/<cat_id>')
def delete_cat(cat_id):
    if not check_auth(): return Response('Login Required', 401)
    categories.delete_one({"_id": ObjectId(cat_id)})
    return redirect(url_for('admin_cats'))

@app.route('/admin/movie/edit/<movie_id>', methods=['GET', 'POST'])
def admin_edit_movie(movie_id):
    if not check_auth(): return Response('Login Required', 401)
    
    if request.method == 'POST':
        update_data = {
            "title": request.form.get("title"),
            "category": request.form.get("category"), # ক্যাটাগরি আপডেট
            "language": request.form.get("language"),
            "overview": request.form.get("overview"),
            "poster": request.form.get("poster"),
            "backdrop": request.form.get("backdrop"),
            "release_date": request.form.get("release_date"),
            "vote_average": request.form.get("vote_average"),
            "type": request.form.get("type"), # টাইপ আপডেট (Movie/Series)
            "updated_at": datetime.utcnow()
        }
        movies.update_one({"_id": ObjectId(movie_id)}, {"$set": update_data})
        return redirect(url_for('admin_home'))
        
    movie = movies.find_one({"_id": ObjectId(movie_id)})
    cat_list = list(categories.find()) # এডিটের জন্য ক্যাটাগরি লিস্ট
    
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_edit)
    return render_template_string(full_html, movie=movie, categories=cat_list, active='dashboard')

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
    
    full_html = admin_base.replace('<!-- CONTENT_GOES_HERE -->', admin_settings)
    return render_template_string(full_html, settings=curr_settings, active='settings')

# API for Admin Panel
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
    if WEBSITE_URL and BOT_TOKEN:
        hook_url = f"{WEBSITE_URL.rstrip('/')}/webhook/{BOT_TOKEN}"
        try: requests.get(f"https://api.telegram.org/bot{BOT_TOKEN}/setWebhook?url={hook_url}")
        except: pass

    port = int(os.environ.get("PORT", 5000))
    app.run(host='0.0.0.0', port=port, debug=True)
