from flask import Flask
from telegram import Update, Bot
from telegram.ext import CommandHandler, CallbackContext, Updater
import json
import threading
import time
import requests
import os

app = Flask(__name__)

TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
bot = Bot(TELEGRAM_TOKEN)

UID_FILE = "uids.json"

def load_uids():
    if not os.path.exists(UID_FILE):
        with open(UID_FILE, "w") as f:
            json.dump([], f)
    with open(UID_FILE, "r") as f:
        return json.load(f)

def save_uids(uids):
    with open(UID_FILE, "w") as f:
        json.dump(uids, f, indent=4)

def check_uid_status(uid):
    try:
        url = f"https://graph.facebook.com/{uid}?fields=id&access_token=EAAG..."
        response = requests.get(url, timeout=5)
        return response.status_code == 200
    except:
        return False

def check_loop():
    while True:
        uids = load_uids()
        for item in uids:
            if check_uid_status(item['uid']):
                msg = f"🍀 UID {item['uid']} đã LIVE ✅\n👤 {item['name']}"
                bot.send_message(chat_id=item['chat_id'], text=msg)
                uids.remove(item)
                save_uids(uids)
        time.sleep(30)

@app.route("/")
def index():
    return "FB UID Checker Bot is running!"

def start(update: Update, context: CallbackContext):
    chat_id = update.message.chat_id
    update.message.reply_text(f"🍀 Chat ID của bạn là: {chat_id}")

def adduid(update: Update, context: CallbackContext):
    args = context.args
    if len(args) < 2:
        update.message.reply_text("Cú pháp: /adduid uid name")
        return
    uid, name = args[0], " ".join(args[1:])
    uids = load_uids()
    uids.append({"uid": uid, "name": name, "chat_id": update.message.chat_id})
    save_uids(uids)
    update.message.reply_text(f"Đã thêm UID {uid} - {name}")

def deluid(update: Update, context: CallbackContext):
    args = context.args
    if len(args) < 1:
        update.message.reply_text("Cú pháp: /deluid uid")
        return
    uid = args[0]
    uids = load_uids()
    uids = [item for item in uids if item['uid'] != uid]
    save_uids(uids)
    update.message.reply_text(f"Đã xóa UID {uid}")

def run_bot():
    updater = Updater(TELEGRAM_TOKEN)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("adduid", adduid))
    dp.add_handler(CommandHandler("deluid", deluid))

    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    threading.Thread(target=check_loop, daemon=True).start()
    threading.Thread(target=run_bot, daemon=True).start()
    app.run(host="0.0.0.0", port=5000)
