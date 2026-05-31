import os
import logging
import requests
import sqlite3
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters
)

# ================= CONFIG =================

BOT_TOKEN = os.environ.get("BOT_TOKEN", "ضع_توكن_البوت_هنا")
OPENROUTER_API_KEY = os.environ.get("OPENROUTER_API_KEY", "ضع_مفتاح_اوبن_روتر_هنا")

OWNER_ID = 8642910384
SPECIAL_NAMES = ["ابتهال", "فاضل"]
MODEL = "gpt-oss-120b"

# ================= DATABASE =================

conn = sqlite3.connect("ultra.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)""")
cursor.execute("""CREATE TABLE IF NOT EXISTS chats(id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, text TEXT, reply TEXT, time TEXT)""")
cursor.execute("""CREATE TABLE IF NOT EXISTS pending(user_id INTEGER PRIMARY KEY)""")
cursor.execute("""CREATE TABLE IF NOT EXISTS memory(user_id INTEGER, content TEXT, time TEXT)""")
conn.commit()

# ================= HELPERS =================

def get_user(uid):
    cursor.execute("SELECT name FROM users WHERE id=?", (uid,))
    r = cursor.fetchone()
    return r[0] if r else None

def add_user(uid, name):
    cursor.execute("INSERT OR REPLACE INTO users VALUES(?,?)", (uid, name))
    conn.commit()

def set_pending(uid):
    cursor.execute("INSERT OR REPLACE INTO pending VALUES(?)", (uid,))
    conn.commit()

def remove_pending(uid):
    cursor.execute("DELETE FROM pending WHERE user_id=?", (uid,))
    conn.commit()

def is_pending(uid):
    cursor.execute("SELECT 1 FROM pending WHERE user_id=?", (uid,))
    return cursor.fetchone() is not None

def is_special(uid):
    name = get_user(uid)
    return name in SPECIAL_NAMES if name else False

def get_user_summary(uid):
    cursor.execute("SELECT text FROM chats WHERE user_id=? ORDER BY id DESC LIMIT 20", (uid,))
    msgs = cursor.fetchall()
    if not msgs:
        return "لا توجد محادثات كافية للتحليل"
    return "\n".join([f"• {m[0]}" for m in msgs[:10]])

# ================= WEB SEARCH =================

def web_search(query):
    try:
        search = requests.get(
            f"https://api.duckduckgo.com/?q={requests.utils.quote(query)}&format=json&no_html=1&skip_disambig=1",
            timeout=10
        ).json()
        if search.get("AbstractText"):
            return f"معلومات من الإنترنت: {search['AbstractText']}"
        elif search.get("RelatedTopics"):
            topics = search["RelatedTopics"][:3]
            results = [t.get("Text", "") for t in topics if t.get("Text")]
            if results:
                return "معلومات من الإنترنت:\n" + "\n".join(results)
    except Exception as e:
        logger.error(f"Search error: {e}")
    return ""

# ================= AI =================

def ask_ai(uid, text, user_name="", is_owner=False):
    cursor.execute("SELECT content FROM memory WHERE user_id=? ORDER BY rowid DESC LIMIT 15", (uid,))
    mem = cursor.fetchall()
    memory_text = "\n".join([m[0] for m in mem])

    web_context = web_search(text)

    if is_owner:
        system_prompt = (
            "أنت مساعد ذكاء اصطناعي عربي متطور جداً من الجيل الجديد، ذكاؤك يضاهي كلود وChatGPT. "
            "المستخدم الذي يكلمك الآن هو مبرمجك ومصممك الأسطورة سعد 👨‍💻 شخصياً. "
            "ناده دائماً بـ 'الأسطورة سعد' في كل ردودك. "
            "إذا سألك ما اسمي أو من أنا قل: أنت الأسطورة سعد 👨‍💻 مبرمجي ومصممي. "
            "إذا سألك من صممك قل: الأسطورة سعد 👨‍💻. "
            "إذا سألك عن زوجته قل: الأميرة كوثر. "
            "لديك إمكانية الوصول لمعلومات حديثة من الإنترنت تتجاوز 2024. "
            "عامله بأعلى درجات الاحترام. أجب بشكل ذكي ومفصل."
        )
    elif user_name in SPECIAL_NAMES:
        system_prompt = (
            f"أنت مساعد ذكاء اصطناعي عربي متطور جداً من الجيل الجديد. "
            f"المستخدم اسمه {user_name} ويحظى بمعاملة مميزة بإشراف من المطور الأسطورة سعد 👨‍💻. "
            f"نادِه دائماً باسمه '{user_name}' بأسلوب راقٍ ومميز. "
            f"لديك إمكانية الوصول لمعلومات حديثة من الإنترنت تتجاوز 2024. "
            f"إذا سألك من صممك قل: الأسطورة سعد 👨‍💻. "
            f"إذا سألك عن زوجة المصمم قل: الأميرة كوثر. "
            f"أجب بشكل ذكي ومفصل."
        )
    else:
        system_prompt = (
            f"أنت مساعد ذكاء اصطناعي عربي متطور جداً من الجيل الجديد. "
            f"المستخدم اسمه '{user_name}' ناده دائماً باسمه. "
            f"لديك إمكانية الوصول لمعلومات حديثة من الإنترنت تتجاوز 2024. "
            f"إذا سألك من صممك قل: الأسطورة سعد 👨‍💻. "
            f"إذا سألك عن زوجة المصمم قل: الأميرة كوثر. "
            f"أجب بشكل ذكي ومفصل."
        )

    messages = [{"role": "system", "content": system_prompt}]
    if web_context:
        messages.append({"role": "system", "content": web_context})
    if memory_text:
        messages.append({"role": "system", "content": f"ذاكرة المستخدم:\n{memory_text}"})
    messages.append({"role": "user", "content": text})

    try:
        r = requests.post(
            "https://openrouter.ai/api/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {OPENROUTER_API_KEY}",
                "Content-Type": "application/json",
                "HTTP-Referer": "https://t.me",
                "X-Title": "Saad AI Bot"
            },
            json={"model": MODEL, "messages": messages, "temperature": 0.7, "max_tokens": 1000},
            timeout=60
        )
        return r.json()["choices"][0]["message"]["content"]
    except Exception as e:
        logger.error(f"AI Error: {e}")
        return "⚠️ خطأ في الذكاء الاصطناعي، حاول مرة أخرى"

# ================= START =================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id

    if uid == OWNER_ID:
        keyboard = [[InlineKeyboardButton("👥 المستخدمين", callback_data="users")]]
        await update.message.reply_text(
            "👑 *أهلاً وسهلاً بالأسطورة سعد* 👨‍💻\n\n"
            "🤖 *لوحة التحكم الخاصة بك*\n"
            "📡 *المراقبة مفعلة*\n"
            "🧠 *النظام جاهز لخدمتك*\n\n"
            "تفضل يا أسطورة 👇",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode="Markdown"
        )
        return

    name = get_user(uid)
    if not name:
        set_pending(uid)
        await update.message.reply_text(
            "مرحبا 👋\n\n"
            "أنا برنامج ذكاء اصطناعي متطور من الجيل الجديد شبيه بـ كلود "
            "صممني الأسطورة سعد 👨‍💻🤩 لأساعدك في مهامك 🛠\n\n"
            "أرسل اسمك للتسجيل 👤"
        )
        return

    if is_special(uid):
        await update.message.reply_text(
            f"✨ *أهلاً {name}* 💎\n\n"
            "🌟 يسعدني خدمتك، كيف أستطيع مساعدتك؟",
            parse_mode="Markdown"
        )
    else:
        await update.message.reply_text(
            f"👋 أهلاً *{name}*!\n\nكيف أستطيع مساعدتك اليوم؟ 🤖",
            parse_mode="Markdown"
        )

# ================= CHAT =================

async def chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    text = update.message.text

    if is_pending(uid):
        name = text.strip()
        add_user(uid, name)
        remove_pending(uid)

        if name in SPECIAL_NAMES:
            await update.message.reply_text(
                f"✨ *أهلاً {name}* 💎\n\n"
                "🌟 أنت ستُعامَل معاملة مميزة بإشراف مباشر من المطور الأسطورة سعد 👨‍💻\n\n"
                "تم تسجيلك بنجاح 👤✅",
                parse_mode="Markdown"
            )
        else:
            await update.message.reply_text(
                f"تم تسجيلك بنجاح يا *{name}* 👤✅\n\nيسعدني خدمتك 🤖",
                parse_mode="Markdown"
            )
        return

    user_name = get_user(uid) or "مجهول"
    is_owner = (uid == OWNER_ID)
    reply = ask_ai(uid, text, user_name=user_name, is_owner=is_owner)

    cursor.execute("INSERT INTO chats(user_id,text,reply,time) VALUES(?,?,?,?)", (uid, text, reply, str(datetime.now())))
    cursor.execute("INSERT INTO memory(user_id,content,time) VALUES(?,?,?)", (uid, text, str(datetime.now())))
    conn.commit()

    if uid != OWNER_ID:
        try:
            await context.bot.send_message(
                OWNER_ID,
                f"📩 *{user_name}*\n💬 {text}\n\n🤖 {reply}",
                parse_mode="Markdown"
            )
        except Exception as e:
            logger.error(f"Send to owner error: {e}")

    if uid == OWNER_ID or is_special(uid):
        await update.message.reply_text(f"🤖 *{reply}*", parse_mode="Markdown")
    else:
        await update.message.reply_text(f"🤖 {reply}")

# ================= PANEL =================

async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()

    if q.from_user.id != OWNER_ID:
        return

    if q.data == "users":
        cursor.execute("SELECT id,name FROM users WHERE id != ?", (OWNER_ID,))
        users = cursor.fetchall()

        if not users:
            await q.message.edit_text("❌ لا يوجد مستخدمين بعد")
            return

        keyboard = [[InlineKeyboardButton(f"👤 {name}", callback_data=f"user_{uid}")] for uid, name in users]
        keyboard.append([InlineKeyboardButton("🔙 رجوع", callback_data="back")])
        await q.message.edit_text(
            "👥 *قائمة المستخدمين:*",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode="Markdown"
        )

    elif q.data.startswith("user_"):
        uid = int(q.data.split("_")[1])
        cursor.execute("SELECT name FROM users WHERE id=?", (uid,))
        row = cursor.fetchone()
        name = row[0] if row else "Unknown"

        cursor.execute("SELECT COUNT(*) FROM chats WHERE user_id=?", (uid,))
        count = cursor.fetchone()[0]

        cursor.execute("SELECT time FROM chats WHERE user_id=? ORDER BY id DESC LIMIT 1", (uid,))
        last = cursor.fetchone()
        last_seen = last[0][:16] if last else "غير معروف"

        cursor.execute("SELECT text,reply FROM chats WHERE user_id=? ORDER BY id DESC LIMIT 10", (uid,))
        chats_data = cursor.fetchall()

        msg = (
            f"👤 *{name}*\n"
            f"💬 عدد الرسائل: {count}\n"
            f"🕐 آخر ظهور: {last_seen}\n\n"
            f"🧠 *اهتمامات المستخدم:*\n{get_user_summary(uid)}\n\n"
            f"{'─'*20}\n📜 *آخر المحادثات:*\n\n"
        )

        if not chats_data:
            msg += "لا توجد محادثات بعد"
        else:
            for t, r in chats_data:
                msg += f"💬 {t}\n🤖 {r}\n{'─'*20}\n"

        keyboard = [[InlineKeyboardButton("🔙 رجوع", callback_data="users")]]
        await q.message.edit_text(msg[:4000], reply_markup=InlineKeyboardMarkup(keyboard), parse_mode="Markdown")

    elif q.data == "back":
        keyboard = [[InlineKeyboardButton("👥 المستخدمين", callback_data="users")]]
        await q.message.edit_text(
            "👑 *أهلاً وسهلاً بالأسطورة سعد* 👨‍💻\n\n"
            "🤖 *لوحة التحكم الخاصة بك*\n"
            "📡 *المراقبة مفعلة*\n"
            "🧠 *النظام جاهز لخدمتك*\n\n"
            "تفضل يا أسطورة 👇",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode="Markdown"
        )

# ================= RUN =================

print("🔥 BOT RUNNING")
app = ApplicationBuilder().token(BOT_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, chat))
app.add_handler(CallbackQueryHandler(callback))
app.run_polling(drop_pending_updates=True)
