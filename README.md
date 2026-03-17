 import os
import json
from datetime import datetime
from flask import Flask, request, jsonify, render_template_string, session

app = Flask(__name__)
# Secret key is required to use Flask Sessions (it saves the 'state' of the chat)
app.config['SECRET_KEY'] = 'physics_super_secret_99'

# --- FILE PATHS ---
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
JSON_PATH = os.path.join(BASE_DIR, 'physics_data.json')
HISTORY_PATH = os.path.join(BASE_DIR, 'search_history.txt')

# --- DATABASE SETUP ---
# I've added multiple entries for "Reflecting Surface" to demonstrate the change
def setup_database():
    data = {
        "entries": [
            {"quantity": "Reflecting Surface (Definition)", "formula": "Angle of Incidence (i) = Angle of Reflection (r)", "dimension": "Conceptual Law"},
            {"quantity": "Reflecting Surface (Regular)", "formula": "Parallel rays remain parallel", "dimension": "Smooth Surface"},
            {"quantity": "Reflecting Surface (Diffuse)", "formula": "Rays scatter in different directions", "dimension": "Rough Surface"},
            {"quantity": "Force", "formula": "F = m × a", "dimension": "[MLT⁻²]"},
            {"quantity": "Pressure", "formula": "P = F / A", "dimension": "[ML⁻¹T⁻²]"},
            {"quantity": "Linear Momentum", "formula": "p = m × v", "dimension": "[MLT⁻¹]"}
        ]
    }
    with open(JSON_PATH, 'w', encoding='utf-8') as f:
        json.dump(data, f)

if not os.path.exists(JSON_PATH):
    setup_database()

# --- FRONTEND UI ---
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        .glass { background: rgba(255, 255, 255, 0.03); backdrop-filter: blur(15px); border: 1px solid rgba(255, 255, 255, 0.1); }
        .neon-glow { box-shadow: 0 0 25px rgba(129, 140, 248, 0.4); }
        body { background-color: #0b0e14; height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; color: white; overflow: hidden; font-family: sans-serif; }
    </style>
</head>
<body>
    <div class="w-full max-w-[360px] h-full flex flex-col items-center justify-between p-8">
        <div class="glass neon-glow w-full aspect-square rounded-[40px] flex items-center justify-center p-6 mt-10">
            <div id="ai-text" class="w-full text-center">
                <h1 class="text-2xl font-serif italic text-gray-500">standing by...</h1>
            </div>
        </div>
        <div class="w-full pb-12">
            <form onsubmit="event.preventDefault(); askAI(false);" class="relative">
                <input id="user-msg" type="text" placeholder="Search Physics..." 
                    class="w-full h-14 glass rounded-full px-6 outline-none text-center focus:border-indigo-500 text-white">
                <button type="submit" class="absolute right-2 top-2 bg-indigo-600 w-10 h-10 rounded-full flex items-center justify-center">
                    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M5 10l7-7m0 0l7 7m-7-7v18" stroke-width="2" stroke-linecap="round"></path></svg>
                </button>
            </form>
        </div>
    </div>
    <script>
        let lastSearch = "";

        async function askAI(isRetry = false) {
            const input = document.getElementById('user-msg');
            const display = document.getElementById('ai-text');
            
            const query = isRetry ? lastSearch : input.value.trim();
            if(!query) return;
            
            lastSearch = query;
            display.innerHTML = '<p class="text-indigo-400 animate-pulse text-sm">Searching...</p>';
            
            try {
                const response = await fetch('/api/chat', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ message: query, retry: isRetry })
                });
                const data = await response.json();
                display.innerHTML = data.reply;
            } catch (e) { display.innerHTML = '<p class="text-red-400">Error</p>'; }
            if(!isRetry) input.value = '';
        }
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route('/api/chat', methods=['POST'])
def chat():
    data = request.json
    user_query = data.get("message", "").lower().strip()
    is_retry = data.get("retry", False)

    # 1. READ DATABASE
    with open(JSON_PATH, 'r', encoding='utf-8') as f:
        db = json.load(f)
    
    # 2. FILTER ALL MATCHES (Finds everything related to the topic)
    matches = [item for item in db['entries'] if user_query in item['quantity'].lower()]
            
    if matches:
        # Use session to remember the index of the last answer shown
        session_key = f"idx_{user_query.replace(' ', '_')}"
        last_idx = session.get(session_key, -1)
        
        # If it's a retry, move to the next item in the match list
        if is_retry:
            current_idx = (last_idx + 1) % len(matches)
        else:
            current_idx = 0
            
        session[session_key] = current_idx
        result = matches[current_idx]

        reply = f'''
            <div class="flex flex-col items-center gap-2">
                <div class="text-indigo-400 font-black text-[10px] uppercase tracking-widest">{result['quantity']}</div>
                <div class="text-[10px] text-gray-400 italic font-serif opacity-70 mb-2">{result['formula']}</div>
                <div class="bg-indigo-500/10 px-6 py-4 rounded-2xl border border-indigo-500/30">
                    <span class="text-2xl font-mono text-white tracking-widest font-bold">{result['dimension']}</span>
                </div>
                <button onclick="askAI(true)" class="mt-4 text-[10px] text-indigo-400 underline opacity-60 hover:opacity-100">
                    Not satisfied? Show another.
                </button>
            </div>
        '''
    else:
        reply = '<p class="text-gray-500">No matches found for this topic.</p>'
        
    return jsonify({"reply": reply})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
    
