import os
import asyncio
import json
import time
import requests
from telethon import TelegramClient, events, Button

# =========================
# Variabili d'ambiente
# =========================
API_ID = int(os.environ.get("API_ID"))
API_HASH = os.environ.get("API_HASH")
PHONE = os.environ.get("PHONE")
OPENROUTER_KEY = os.environ.get("OPENROUTER_KEY")

client = TelegramClient("session", API_ID, API_HASH)

# =========================
# Dati utenti
# =========================
DATA_FILE = "user_data.json"
if not os.path.exists(DATA_FILE):
    with open(DATA_FILE, "w") as f:
        json.dump({}, f)

def load_data():
    with open(DATA_FILE, "r") as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=2)

def add_points(user_id, pts):
    data = load_data()
    user = data.setdefault(str(user_id), {})
    user["points"] = user.get("points", 0) + pts
    save_data(data)
    return user["points"]

def get_points(user_id):
    data = load_data()
    return data.get(str(user_id), {}).get("points", 0)

start_time = time.time()
greeted_users = set()

# =========================
# Comandi base
# =========================
@client.on(events.NewMessage(outgoing=True, pattern=r"\.ping"))
async def ping(event):
    await event.edit("🏓 Pong!")

@client.on(events.NewMessage(outgoing=True, pattern=r"\.help"))
async def help_cmd(event):
    text = (
        "Comandi disponibili:\n"
        ".ping, .help, .echo <testo>, .uptime, .myid\n"
        ".balance, .daily, .ask <messaggio>\n"
        ".mute <@utente> <tempo>, .unmute <@utente>"
    )
    await event.edit(text)

@client.on(events.NewMessage(outgoing=True, pattern=r"\.echo (.+)"))
async def echo(event):
    await event.edit(event.pattern_match.group(1))

@client.on(events.NewMessage(outgoing=True, pattern=r"\.uptime"))
async def uptime(event):
    elapsed = int(time.time() - start_time)
    h = elapsed // 3600
    m = (elapsed % 3600) // 60
    s = elapsed % 60
    await event.edit(f"⏱ Bot attivo da {h}h {m}m {s}s")

@client.on(events.NewMessage(outgoing=True, pattern=r"\.myid"))
async def myid(event):
    me = await client.get_me()
    await event.edit(f"👤 Il tuo ID: {me.id}")

# =========================
# Punti e daily
# =========================
@client.on(events.NewMessage(outgoing=True, pattern=r"\.balance"))
async def balance(event):
    user_id = event.sender_id
    pts = get_points(user_id)
    await event.edit(f"💰 Hai {pts} punti disponibili.")

@client.on(events.NewMessage(outgoing=True, pattern=r"\.daily"))
async def daily(event):
    user_id = event.sender_id
    data = load_data()
    user = data.setdefault(str(user_id), {})
    last = user.get("last_daily", 0)
    now = time.time()
    if now - last < 24*3600:
        await event.edit("⏳ Hai già ritirato i punti giornalieri, riprova domani.")
        return
    pts = 50
    user["points"] = user.get("points", 0) + pts
    user["last_daily"] = now
    save_data(data)
    await event.edit(f"🎁 Hai ricevuto {pts} punti! 💰 Totale: {user['points']}pt")

# =========================
# Comando AI .ask
# =========================
@client.on(events.NewMessage(outgoing=True, pattern=r"\.ask (.+)"))
async def ask_ai(event):
    query = event.pattern_match.group(1)
    msg = await event.respond("🤖 Sto pensando...")
    try:
        headers = {"Authorization": f"Bearer {OPENROUTER_KEY}"}
        json_data = {"model": "gpt-4o-mini", "messages":[{"role":"user","content":query}], "temperature":0.7}
        response = requests.post("https://openrouter.ai/api/v1/chat/completions", headers=headers, json=json_data, timeout=15)
        response.raise_for_status()
        data = response.json()
        ai_text = data["choices"][0]["message"]["content"]
        await msg.edit(f"🤖 Risposta AI:\n{ai_text}")
    except Exception as e:
        await msg.edit(f"❌ Errore nella richiesta AI:\n{e}")

# =========================
# Mute / Unmute
# =========================
muted_users = {}

@client.on(events.NewMessage(outgoing=True, pattern=r"\.mute (.+) (\d+)"))
async def mute(event):
    mention = event.pattern_match.group(1)
    seconds = int(event.pattern_match.group(2))
    # Cerca ID utente
    try:
        user = await client.get_entity(mention)
        muted_users[user.id] = time.time() + seconds
        await event.edit(f"😎 {mention} è stato silenziato per {seconds} secondi.")
    except Exception as e:
        await event.edit(f"❌ Errore: {e}")

@client.on(events.NewMessage(outgoing=True, pattern=r"\.unmute (.+)"))
async def unmute(event):
    mention = event.pattern_match.group(1)
    try:
        user = await client.get_entity(mention)
        if user.id in muted_users:
            del muted_users[user.id]
            await event.edit(f"😎 {mention} è stato sbloccato.")
        else:
            await event.edit("❌ Utente non era silenziato.")
    except Exception as e:
        await event.edit(f"❌ Errore: {e}")

# =========================
# Welcome con GIF
# =========================
WELCOME_MSG = "💀 Benvenuto {name}!\nUsa .ask per parlare con me.\n— BLACK_DAGGER"
GIF_URL = "https://media.giphy.com/media/3oEjI6SIIHBdRxXI40/giphy.gif"

@client.on(events.NewMessage(incoming=True))
async def welcome(event):
    if not event.is_private:
        return
    sender = await event.get_sender()
    user_id = sender.id
    name = sender.first_name or "utente"
    if user_id in greeted_users:
        return
    await client.send_file(event.chat_id, GIF_URL, caption=WELCOME_MSG.format(name=name))
    greeted_users.add(user_id)

# =========================
# Avvio bot
# =========================
async def start_bot():
    while True:
        try:
            await client.start(phone=PHONE)
            print("🔥 USERBOT ONLINE")
            await client.run_until_disconnected()
        except Exception as e:
            print(f"⚠️ Errore: {e}")
            await asyncio.sleep(5)

if __name__ == "__main__":
    asyncio.run(start_bot())
