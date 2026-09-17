# -*- coding: utf-8 -*-
"""
ربات انسانی | HUMAN BOT
Python 3.10+
Bale Bot API + SQLite
نسخه گروه‌محور:
- بازی فقط داخل گروه فعال است؛ در PV هیچ دستور بازی اجرا نمی‌شود.
- شمارش گروه‌ها و کاربران واقعی و مبتنی بر دیتابیس است.
- رابط کاربری از Inline Keyboard استفاده می‌کند (نه Reply Keyboard).
- TOKEN را فقط در خط پایین وارد کنید.
"""

TOKEN = "1111829688:oS816lZ4dSAB-eOmSQdrgnhrAm0sMcwCCoU"
API = "https://tapi.bale.ai/bot"

import json
import logging
import random
import sqlite3
import threading
import time
import traceback
import urllib.parse
import urllib.request
from datetime import datetime, timedelta

DB_FILE = "human_bot.db"
POLL_TIMEOUT = 35
GROUP_TYPES = {"group", "supergroup"}

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s"
)

db_lock = threading.RLock()
user_locks = {}
user_locks_guard = threading.RLock()


def now_ts():
    return int(time.time())


def fa_num(value):
    s = f"{int(value):,}"
    return s.translate(str.maketrans("0123456789,", "۰۱۲۳۴۵۶۷۸۹٬"))


def fmt_duration(seconds):
    seconds = max(0, int(seconds))
    h, rem = divmod(seconds, 3600)
    m, s = divmod(rem, 60)
    parts = []
    if h:
        parts.append(f"{fa_num(h)} ساعت")
    if m or h:
        parts.append(f"{fa_num(m)} دقیقه")
    parts.append(f"{fa_num(s)} ثانیه")
    return " و ".join(parts)


def conn():
    c = sqlite3.connect(DB_FILE, timeout=30)
    c.row_factory = sqlite3.Row
    return c


def init_db():
    with db_lock:
        c = conn()
        c.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            first_name TEXT DEFAULT '',
            username TEXT DEFAULT '',
            serial INTEGER UNIQUE,
            points INTEGER DEFAULT 0,
            bank INTEGER DEFAULT 0,
            human_calls INTEGER DEFAULT 0,
            hunts INTEGER DEFAULT 0,
            slaves INTEGER DEFAULT 0,
            level INTEGER DEFAULT 1,
            last_human INTEGER DEFAULT 0,
            bank_interest_at INTEGER DEFAULT 0,
            bank_interest_acc REAL DEFAULT 0,
            prison_until INTEGER DEFAULT 0,
            created_at INTEGER DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS groups (
            chat_id INTEGER PRIMARY KEY,
            title TEXT DEFAULT '',
            username TEXT DEFAULT '',
            chat_type TEXT DEFAULT 'group',
            active INTEGER DEFAULT 0,
            installed_at INTEGER DEFAULT 0,
            last_seen INTEGER DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS group_members (
            chat_id INTEGER,
            user_id INTEGER,
            last_seen INTEGER DEFAULT 0,
            PRIMARY KEY(chat_id, user_id)
        );

        CREATE TABLE IF NOT EXISTS transfers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            sender_id INTEGER,
            receiver_id INTEGER,
            amount INTEGER,
            created_at INTEGER
        );

        CREATE TABLE IF NOT EXISTS hunts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            created_at INTEGER,
            last_action INTEGER,
            found_mask INTEGER DEFAULT 0,
            wrong_count INTEGER DEFAULT 0,
            active INTEGER DEFAULT 1,
            kidneys INTEGER DEFAULT 0,
            hearts INTEGER DEFAULT 0,
            eyes INTEGER DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS factories (
            user_id INTEGER PRIMARY KEY,
            level INTEGER DEFAULT 1,
            last_tick INTEGER DEFAULT 0,
            ready INTEGER DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS slaves (
            user_id INTEGER PRIMARY KEY,
            level INTEGER DEFAULT 1,
            stored REAL DEFAULT 0,
            last_tick INTEGER DEFAULT 0,
            name TEXT DEFAULT 'انسان'
        );

        CREATE TABLE IF NOT EXISTS casino_rooms (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            creator_id INTEGER,
            entry INTEGER,
            max_players INTEGER,
            status TEXT DEFAULT 'open',
            created_at INTEGER
        );

        CREATE TABLE IF NOT EXISTS casino_players (
            room_id INTEGER,
            user_id INTEGER,
            PRIMARY KEY(room_id, user_id)
        );

        CREATE TABLE IF NOT EXISTS settings (
            key TEXT PRIMARY KEY,
            value TEXT
        );
        """)
        c.commit()
        c.close()


def db_one(sql, args=()):
    with db_lock:
        c = conn()
        row = c.execute(sql, args).fetchone()
        c.close()
        return row


def db_all(sql, args=()):
    with db_lock:
        c = conn()
        rows = c.execute(sql, args).fetchall()
        c.close()
        return rows


def db_exec(sql, args=()):
    with db_lock:
        c = conn()
        cur = c.execute(sql, args)
        c.commit()
        result = cur.lastrowid
        c.close()
        return result


def user_lock(uid):
    with user_locks_guard:
        if uid not in user_locks:
            user_locks[uid] = threading.RLock()
        return user_locks[uid]


def ensure_user(uid, first_name="", username=""):
    row = db_one("SELECT * FROM users WHERE user_id=?", (uid,))
    if row:
        db_exec(
            "UPDATE users SET first_name=?, username=? WHERE user_id=?",
            (first_name or "", username or "", uid)
        )
        return
    with db_lock:
        c = conn()
        row = c.execute("SELECT COALESCE(MAX(serial),0)+1 AS n FROM users").fetchone()
        serial = int(row["n"])
        c.execute(
            """INSERT INTO users
            (user_id,first_name,username,serial,created_at)
            VALUES (?,?,?,?,?)""",
            (uid, first_name or "", username or "", serial, now_ts())
        )
        c.commit()
        c.close()


def ensure_group(chat):
    if not chat:
        return
    cid = int(chat.get("id"))
    title = chat.get("title", "")
    username = chat.get("username", "")
    typ = chat.get("type", "group")
    existing = db_one("SELECT chat_id FROM groups WHERE chat_id=?", (cid,))
    if existing:
        db_exec(
            """UPDATE groups
               SET title=?,username=?,chat_type=?,last_seen=?
               WHERE chat_id=?""",
            (title, username, typ, now_ts(), cid)
        )
    else:
        db_exec(
            """INSERT INTO groups
               (chat_id,title,username,chat_type,active,last_seen)
               VALUES (?,?,?,?,0,?)""",
            (cid, title, username, typ, now_ts())
        )


def record_member(chat_id, user):
    if not user or not user.get("id"):
        return
    uid = int(user["id"])
    ensure_user(uid, user.get("first_name", ""), user.get("username", ""))
    db_exec(
        """INSERT INTO group_members(chat_id,user_id,last_seen)
           VALUES(?,?,?)
           ON CONFLICT(chat_id,user_id)
           DO UPDATE SET last_seen=excluded.last_seen""",
        (chat_id, uid, now_ts())
    )


def group_is_active(chat_id):
    row = db_one("SELECT active FROM groups WHERE chat_id=?", (chat_id,))
    return bool(row and row["active"])


def count_groups():
    row = db_one("SELECT COUNT(*) AS n FROM groups WHERE active=1")
    return int(row["n"])


def count_users():
    row = db_one("SELECT COUNT(*) AS n FROM users")
    return int(row["n"])


def api(method, data=None):
    if not TOKEN.strip():
        raise RuntimeError("TOKEN خالی است. مقدار TOKEN را در ابتدای فایل وارد کنید.")
    url = f"{API}{TOKEN}/{method}"
    body = urllib.parse.urlencode(data or {}).encode("utf-8")
    req = urllib.request.Request(url, data=body)
    with urllib.request.urlopen(req, timeout=60) as r:
        raw = r.read().decode("utf-8")
    result = json.loads(raw)
    return result


def send_message(chat_id, text, keyboard=None, reply_to=None):
    data = {"chat_id": chat_id, "text": text}
    if keyboard is not None:
        data["reply_markup"] = json.dumps(keyboard, ensure_ascii=False)
    if reply_to:
        data["reply_to_message_id"] = reply_to
    return api("sendMessage", data)


def edit_message(chat_id, message_id, text, keyboard=None):
    data = {"chat_id": chat_id, "message_id": message_id, "text": text}
    if keyboard is not None:
        data["reply_markup"] = json.dumps(keyboard, ensure_ascii=False)
    return api("editMessageText", data)


def answer_callback(callback_id, text=""):
    return api("answerCallbackQuery", {
        "callback_query_id": callback_id,
        "text": text
    })


def kb(rows):
    return {"inline_keyboard": rows}


def btn(text, data):
    return {"text": text, "callback_data": data}


def main_menu():
    return kb([
        [btn(f"👥 تعداد گروه‌های من : {fa_num(count_groups())}", "noop_groups")],
        [btn(f"👤 تعداد کاربران من : {fa_num(count_users())}", "noop_users")],
        [btn("➕ افزودن من به گروه", "add_group")],
    ])

def private_menu():
    # PV فقط منوی معرفی/افزودن گروه را دارد و هیچ قابلیت بازی در آن فعال نیست.
    return main_menu()


def game_menu():
    return kb([
        [btn("👤 پروفایل", "profile"),
         btn("👨 انسان", "human")],
        [btn("🏦 بانک انسانی", "bank"),
         btn("🎰 کازینو انسانی", "casino")],
        [btn("🔎 شکار انسانی", "hunt"),
         btn("👥 برده انسانی", "slave")],
        [btn("📦 قاچاق انسانی", "contraband"),
         btn("🏭 کارخانه انسانی", "factory")],
        [btn("💸 انتقال انسانی", "transfer_help"),
         btn("📖 راهنما", "help")],
    ])


def group_menu_text():
    return (
        "🤖 ربات انسانی\n\n"
        "به دنیای ربات انسانی خوش آمدید!\n"
        "از منوی زیر بازی را در همین گروه ادامه دهید."
    )


def profile_text(uid):
    u = db_one("SELECT * FROM users WHERE user_id=?", (uid,))
    if not u:
        return "پروفایل پیدا نشد."
    return (
        "👤 پروفایل انسانی\n\n"
        f"نام : {u['first_name'] or 'بدون نام'}\n"
        f"یوزرنیم کاربری : {fa_num(u['serial'])}\n"
        f"انسان پوینت : {fa_num(u['points'])}\n"
        f"موجودی بانک انسانی : {fa_num(u['bank'])}\n"
        f"شکارهای انسانی : {fa_num(u['hunts'])}\n"
        f"برده انسانی : {fa_num(u['slaves'])}\n"
        f"سطح : {fa_num(u['level'])}"
    )


LEVELS = {
    1: (5, "🏦 بانک انسانی"),
    2: (20, "🎰 کازینو انسانی"),
    3: (50, "🔎 شکار انسانی"),
    4: (100, "🏭 کارخانه انسانی"),
    5: (200, "👥 برده انسانی"),
    6: (500, "📦 قاچاق انسانی"),
    7: (1000, "📦 قاچاق انسانی"),
}


def level_from_calls(calls):
    level = 1
    for target in sorted(LEVELS):
        if calls >= LEVELS[target][0]:
            level = min(target + 1, 8)
    return level


def sync_level(uid):
    u = db_one("SELECT human_calls,level FROM users WHERE user_id=?", (uid,))
    if not u:
        return 1
    calculated = level_from_calls(int(u["human_calls"]))
    if calculated > int(u["level"]):
        db_exec("UPDATE users SET level=? WHERE user_id=?", (calculated, uid))
    return calculated
def cooldown_human(uid):
    row = db_one("SELECT last_human FROM users WHERE user_id=?", (uid,))
    if not row:
        return 0
    return max(0, 300 - (now_ts() - int(row["last_human"])))


def do_human(uid):
    with user_lock(uid):
        ensure_not_prison(uid)
        remain = cooldown_human(uid)
        if remain > 0:
            return (
                "⏳ هنوز نمیشه انسان گفت!\n\n"
                f"زمان باقی‌مانده : {fmt_duration(remain)}"
            )
        gain = random.randint(1, 50)
        db_exec(
            """UPDATE users
               SET points=points+?,human_calls=human_calls+1,last_human=?
               WHERE user_id=?""",
            (gain, now_ts(), uid)
        )
        old = db_one("SELECT human_calls FROM users WHERE user_id=?", (uid,))
        new_level = sync_level(uid)
        text = (
            "👨 انسان پیدا شد!\n\n"
            f"➕ دریافت : {fa_num(gain)} انسان پوینت\n"
            f"📞 تعداد انسان : {fa_num(old['human_calls'])}\n"
            f"⭐ سطح : {fa_num(new_level)}"
        )
        if new_level < 8:
            target = LEVELS.get(new_level)
            if target:
                needed = max(0, target[0] - int(old["human_calls"]))
                text += f"\n\n🔓 قابلیت بعدی: {target[1]}\nتعداد باقی‌مانده: {fa_num(needed)}"
        return text


def apply_bank_interest(uid):
    with user_lock(uid):
        u = db_one(
            "SELECT bank,bank_interest_at,bank_interest_acc FROM users WHERE user_id=?",
            (uid,)
        )
        if not u:
            return
        t = int(u["bank_interest_at"] or 0)
        current = now_ts()
        if t == 0:
            db_exec("UPDATE users SET bank_interest_at=? WHERE user_id=?", (current, uid))
            return
        days = (current - t) // 86400
        if days <= 0:
            return
        bank = int(u["bank"])
        # سود ۵٪ روزانه به شکل مرکب؛ فقط برای موجودی بانکی بازی.
        for _ in range(days):
            bank = int(bank * 1.05)
        db_exec(
            "UPDATE users SET bank=?,bank_interest_at=? WHERE user_id=?",
            (bank, t + days * 86400, uid)
        )


def bank_text(uid):
    apply_bank_interest(uid)
    u = db_one("SELECT bank,points,level FROM users WHERE user_id=?", (uid,))
    if not u:
        return "پروفایل پیدا نشد."
    if int(u["level"]) < 2:
        return "🔒 بانک انسانی\n\nبرای باز شدن بانک باید به سطح ۲ برسید."
    return (
        "🏦 بانک انسانی\n\n"
        f"کیف پول : {fa_num(u['points'])} انسان پوینت\n"
        f"موجودی بانک : {fa_num(u['bank'])} انسان پوینت\n"
        "📈 سود روزانه : ۵٪"
    )


def bank_menu():
    return kb([
        [btn("➕ سپرده", "bank_deposit"), btn("➖ برداشت", "bank_withdraw")],
        [btn("🔄 بروزرسانی", "bank"), btn("↩️ برگشت", "back")]
    ])


def level_required(uid, level):
    u = db_one("SELECT level FROM users WHERE user_id=?", (uid,))
    return bool(u and int(u["level"]) >= level)


def ensure_not_prison(uid):
    u = db_one("SELECT prison_until FROM users WHERE user_id=?", (uid,))
    if u and int(u["prison_until"]) > now_ts():
        raise PermissionError(
            f"زندان انسانی فعال است.\nزمان باقی‌مانده: {fmt_duration(int(u['prison_until'])-now_ts())}"
        )
    if u and int(u["prison_until"]) != 0:
        db_exec("UPDATE users SET prison_until=0 WHERE user_id=?", (uid,))


def hunt_start(uid):
    if not level_required(uid, 4):
        return "🔒 شکار انسانی از سطح ۴ باز می‌شود."
    try:
        ensure_not_prison(uid)
    except PermissionError as e:
        return str(e)

    active = db_one(
        "SELECT * FROM hunts WHERE user_id=? AND active=1 ORDER BY id DESC LIMIT 1",
        (uid,)
    )
    if active:
        return hunt_view(uid, int(active["id"]))

    last = db_one(
        "SELECT created_at FROM hunts WHERE user_id=? ORDER BY id DESC LIMIT 1",
        (uid,)
    )
    if last and now_ts() - int(last["created_at"]) < 1800:
        remain = 1800 - (now_ts() - int(last["created_at"]))
        return f"🔎 شکار انسانی\n\nهنوز آماده نیست.\nزمان باقی‌مانده: {fmt_duration(remain)}"

    # ۵ خانه از ۹ خانه دارای هدف مجازی هستند.
    positions = random.sample(range(9), 5)
    mask = sum(1 << p for p in positions)
    hid = db_exec(
        """INSERT INTO hunts(user_id,created_at,last_action,found_mask,wrong_count,active)
           VALUES(?,?,?,?,?,1)""",
        (uid, now_ts(), now_ts(), mask, 0)
    )
    return hunt_view(uid, hid)


def hunt_view(uid, hid):
    h = db_one("SELECT * FROM hunts WHERE id=? AND user_id=?", (hid, uid))
    if not h or not int(h["active"]):
        return "این شکار دیگر فعال نیست."
    found = int(h["found_mask"])
    wrong = int(h["wrong_count"])
    rows = []
    row = []
    for i in range(9):
        if found & (1 << i):
            text = "🧍"
        else:
            text = f"🏠 {fa_num(i+1)}"
        row.append(btn(text, f"hunt:{hid}:{i}"))
        if len(row) == 3:
            rows.append(row)
            row = []
    if row:
        rows.append(row)
    rows.append([btn("↩️ برگشت", "back")])
    return (
        "🔎 شکار انسانی\n\n"
        "یک خانه را انتخاب کن.\n"
        "در این شکار ۵ خانه هدف دارند.\n"
        f"خانه‌های از دست‌رفته: {fa_num(wrong)}/۴"
    ), kb(rows)


def hunt_pick(uid, hid, pos):
    with user_lock(uid):
        try:
            ensure_not_prison(uid)
        except PermissionError as e:
            return str(e), None
        h = db_one("SELECT * FROM hunts WHERE id=? AND user_id=?", (hid, uid))
        if not h or not int(h["active"]):
            return "این شکار فعال نیست.", None

        found = int(h["found_mask"])
        if found & (1 << pos):
            return "این خانه قبلاً پیدا شده است.", None

        mask = int(h["found_mask"])
        target = bool(mask is not None and (mask & (1 << pos)))
        # target mask is stored in found_mask at creation; once a target is
        # found we need a separate immutable target mask. For compatibility,
        # encode target mask in last_action is not safe. We therefore use a
        # deterministic target set stored in settings-style payload below.
        # This branch is replaced by hunt_target table in init_db migration.
        return "خطای داخلی شکار.", None


# Migration for proper hunt target persistence.
def migrate_hunt_targets():
    with db_lock:
        c = conn()
        c.execute("""
        CREATE TABLE IF NOT EXISTS hunt_targets (
            hunt_id INTEGER PRIMARY KEY,
            target_mask INTEGER NOT NULL
        )
        """)
        c.commit()
        c.close()


def hunt_start(uid):
    if not level_required(uid, 4):
        return "🔒 شکار انسانی از سطح ۴ باز می‌شود."
    try:
        ensure_not_prison(uid)
    except PermissionError as e:
        return str(e)

    active = db_one(
        "SELECT * FROM hunts WHERE user_id=? AND active=1 ORDER BY id DESC LIMIT 1",
        (uid,)
    )
    if active:
        return hunt_render(uid, int(active["id"]))

    last = db_one(
        "SELECT created_at FROM hunts WHERE user_id=? ORDER BY id DESC LIMIT 1",
        (uid,)
    )
    if last and now_ts() - int(last["created_at"]) < 1800:
        return f"🔎 شکار انسانی\n\nهنوز آماده نیست.\nزمان باقی‌مانده: {fmt_duration(1800-(now_ts()-int(last['created_at'])))}"

    hid = db_exec(
        """INSERT INTO hunts(user_id,created_at,last_action,found_mask,wrong_count,active)
           VALUES(?,?,?,?,?,1)""",
        (uid, now_ts(), now_ts(), 0, 0)
    )
    target_mask = sum(1 << p for p in random.sample(range(9), 5))
    db_exec(
        "INSERT INTO hunt_targets(hunt_id,target_mask) VALUES(?,?)",
        (hid, target_mask)
    )
    return hunt_render(uid, hid)


def hunt_render(uid, hid):
    h = db_one("SELECT * FROM hunts WHERE id=? AND user_id=?", (hid, uid))
    if not h or not int(h["active"]):
        return "این شکار دیگر فعال نیست.", kb([[btn("↩️ برگشت", "back")]])
    found = int(h["found_mask"])
    wrong = int(h["wrong_count"])
    rows = []
    current = []
    for i in range(9):
        current.append(btn("🧍" if found & (1 << i) else f"🏠 {fa_num(i+1)}",
                           f"hunt:{hid}:{i}"))
        if len(current) == 3:
            rows.append(current)
            current = []
    if current:
        rows.append(current)
    rows.append([btn("↩️ برگشت", "back")])
    return (
        "🔎 شکار انسانی\n\n"
        "یکی از خانه‌ها را انتخاب کن.\n"
        "🎯 در این شکار ۵ خانه هدف دارند.\n"
        f"خانه‌های اشتباه: {fa_num(wrong)}/۴"
    ), kb(rows)


def finish_hunt(uid, hid):
    # پاداش‌های کاملاً مجازی بازی.
    kidneys = 1 if random.random() < 0.95 else 0
    if random.random() < 0.70 and kidneys:
        kidneys += 1
    hearts = 1 if random.random() < 0.60 else 0
    eyes = 1 if random.random() < 0.70 else 0
    if random.random() < 0.40 and eyes:
        eyes += 1

    gain = random.randint(100, 400)
    db_exec(
        """UPDATE hunts SET active=0,kidneys=?,hearts=?,eyes=? WHERE id=?""",
        (kidneys, hearts, eyes, hid)
    )
    db_exec(
        "UPDATE users SET hunts=hunts+1,points=points+? WHERE user_id=?",
        (gain, uid)
    )
    return (
        "✅ شکار با موفقیت تمام شد!\n\n"
        f"💰 پاداش : {fa_num(gain)} انسان پوینت\n"
        f"📦 آیتم مجازی کلیه : {fa_num(kidneys)}\n"
        f"📦 آیتم مجازی قلب : {fa_num(hearts)}\n"
        f"📦 آیتم مجازی چشم : {fa_num(eyes)}\n\n"
        "این آیتم‌ها فقط دارایی‌های داخل بازی هستند."
    )


def hunt_pick(uid, hid, pos):
    with user_lock(uid):
        try:
            ensure_not_prison(uid)
        except PermissionError as e:
            return str(e), None

        h = db_one("SELECT * FROM hunts WHERE id=? AND user_id=?", (hid, uid))
        target = db_one("SELECT target_mask FROM hunt_targets WHERE hunt_id=?", (hid,))
        if not h or not int(h["active"]) or not target:
            return "این شکار فعال نیست.", None

        found = int(h["found_mask"])
        if found & (1 << pos):
            return "این خانه قبلاً پیدا شده است.", None

        target_mask = int(target["target_mask"])
        if target_mask & (1 << pos):
            found |= 1 << pos
            db_exec(
                "UPDATE hunts SET found_mask=?,last_action=? WHERE id=?",
                (found, now_ts(), hid)
            )
            if bin(found & target_mask).count("1") >= 5:
                text = finish_hunt(uid, hid)
                return text, kb([[btn("🔎 شکار دوباره", "hunt")],
                                 [btn("↩️ برگشت", "back")]])
            text, keyboard = hunt_render(uid, hid)
            return "🎯 درست بود!\n\n" + text, keyboard

        wrong = int(h["wrong_count"]) + 1
        if wrong >= 4:
            db_exec(
                "UPDATE hunts SET active=0,wrong_count=?,last_action=? WHERE id=?",
                (wrong, now_ts(), hid)
            )
            # احتمال کشف فعالیت: ۵۰٪ و سپس احتمال زندان ۳۰٪
            prison = ""
            if random.random() < 0.50 and random.random() < 0.30:
                duration = random.randint(10, 60) * 60
                db_exec(
                    "UPDATE users SET prison_until=? WHERE user_id=?",
                    (now_ts()+duration, uid)
                )
                prison = f"\n\n🚨 فعالیت شناسایی شد!\n🔒 زندان انسانی: {fmt_duration(duration)}"
            return "❌ شکار شکست خورد!\n\nخانه‌های اشتباه به حد مجاز رسیدند." + prison, \
                   kb([[btn("↩️ برگشت", "back")]])

        db_exec(
            "UPDATE hunts SET wrong_count=?,last_action=? WHERE id=?",
            (wrong, now_ts(), hid)
        )
        text, keyboard = hunt_render(uid, hid)
        return "❌ این خانه خالی بود!\n\n" + text, keyboard


SLAVE_CAP = {i: i * 100 for i in range(1, 21)}
SLAVE_COST = {i: i * 200 for i in range(1, 20)}


def slave_sync(uid):
    s = db_one("SELECT * FROM slaves WHERE user_id=?", (uid,))
    if not s:
        return
    last = int(s["last_tick"] or now_ts())
    elapsed = max(0, now_ts() - last)
    if elapsed <= 0:
        return
    lvl = int(s["level"])
    rate = lvl  # points/minute
    earned = elapsed / 60.0 * rate
    cap = SLAVE_CAP[lvl]
    stored = min(cap, float(s["stored"]) + earned)
    db_exec(
        "UPDATE slaves SET stored=?,last_tick=? WHERE user_id=?",
        (stored, now_ts(), uid)
    )


def slave_start(uid):
    if not level_required(uid, 6):
        return "🔒 برده انسانی از سطح ۶ باز می‌شود."
    row = db_one("SELECT * FROM slaves WHERE user_id=?", (uid,))
    if not row:
        db_exec(
            "INSERT INTO slaves(user_id,level,stored,last_tick,name) VALUES(?,?,?,?,?)",
            (uid, 1, 0, now_ts(), "انسان")
        )
    slave_sync(uid)
    s = db_one("SELECT * FROM slaves WHERE user_id=?", (uid,))
    return (
        "👥 برده انسانی\n\n"
        f"نام : {s['name']}\n"
        f"سطح : {fa_num(s['level'])}\n"
        f"ظرفیت : {fa_num(SLAVE_CAP[int(s['level'])])}\n"
        f"پوینت آماده دریافت : {fa_num(int(s['stored']))}\n"
        f"تولید : {fa_num(int(s['level']))} پوینت در دقیقه"
    )
    def slave_menu():
    return kb([
        [btn("⬆️ ارتقا", "slave_upgrade"),
         btn("💰 دریافت پوینت", "slave_claim")],
        [btn("✏️ تغییر نام", "slave_rename")],
        [btn("↩️ برگشت", "back")]
    ])


def factory_start(uid):
    if not level_required(uid, 5):
        return "🔒 کارخانه انسانی از سطح ۵ باز می‌شود."
    row = db_one("SELECT * FROM factories WHERE user_id=?", (uid,))
    if not row:
        db_exec(
            "INSERT INTO factories(user_id,level,last_tick,ready) VALUES(?,?,?,0)",
            (uid, 1, now_ts())
        )
    else:
        # تولید واقعی بر اساس فاصله زمانی؛ مقدار آماده ذخیره می‌شود.
        f = db_one("SELECT * FROM factories WHERE user_id=?", (uid,))
        interval = factory_interval(int(f["level"]))
        elapsed = max(0, now_ts() - int(f["last_tick"]))
        produced = elapsed // interval
        if produced:
            db_exec(
                "UPDATE factories SET ready=ready+?,last_tick=last_tick+? WHERE user_id=?",
                (produced, produced*interval, uid)
            )
    f = db_one("SELECT * FROM factories WHERE user_id=?", (uid,))
    interval = factory_interval(int(f["level"]))
    remaining = max(0, interval - (now_ts()-int(f["last_tick"])))
    return (
        "🏭 کارخانه انسانی\n\n"
        f"سطح کارخانه : {fa_num(f['level'])}\n"
        f"تولید هر : {fmt_duration(interval)}\n"
        f"انسان آماده دریافت : {fa_num(f['ready'])}\n"
        f"تولید بعدی : {fmt_duration(remaining)}"
    )


def factory_interval(level):
    values = {1:43200,2:28800,3:21600,4:14400,5:7200,6:3600,7:1800}
    return values.get(level, 1800)


def factory_menu():
    return kb([
        [btn("📥 دریافت انسان", "factory_claim"),
         btn("⬆️ ارتقای کارخانه", "factory_upgrade")],
        [btn("↩️ برگشت", "back")]
    ])


FACTORY_COST = {1:20000,2:50000,3:75000,4:100000,5:120000,6:150000}


def contraband_text(uid):
    if not level_required(uid, 7):
        return "🔒 قاچاق انسانی از سطح ۷ باز می‌شود.", None
    # فقط آیتم‌های مجازی بازی.
    h = db_all("""
        SELECT COALESCE(SUM(kidneys),0) k, COALESCE(SUM(hearts),0) h,
               COALESCE(SUM(eyes),0) e
        FROM hunts WHERE user_id=?
    """, (uid,))[0]
    k, heart, eye = int(h["k"]), int(h["h"]), int(h["e"])
    total = k*1000 + heart*3000 + eye*2000
    return (
        "📦 انبار قاچاق انسانی\n\n"
        "⚠️ همه موارد زیر فقط آیتم‌های مجازی بازی هستند.\n\n"
        f"🟤 کلیه : {fa_num(k)}\n"
        f"❤️ قلب : {fa_num(heart)}\n"
        f"👁️ چشم : {fa_num(eye)}\n\n"
        f"💰 ارزش مجازی کل : {fa_num(total)}"
    ), kb([[btn("🔄 بروزرسانی", "contraband")],
           [btn("↩️ برگشت", "back")]])


def casino_text(uid):
    if not level_required(uid, 3):
        return "🔒 کازینو انسانی از سطح ۳ باز می‌شود.", None
    return (
        "🎰 کازینو انسانی\n\n"
        "یک اتاق بساز و اعضای گروه را دعوت کن.\n"
        "ورودی از ۱۰۰ انسان پوینت شروع می‌شود.\n"
        "تعداد بازیکن: ۲ تا ۶ نفر."
    ), kb([
        [btn("🎰 ساخت کازینو", "casino_create")],
        [btn("↩️ برگشت", "back")]
    ])


def help_text():
    return (
        "📖 راهنمای ربات انسانی\n\n"
        "👨 انسان — هر ۵ دقیقه یک بار\n"
        "🏦 بانک — باز شدن در سطح ۲ و سود روزانه ۵٪\n"
        "🎰 کازینو — سطح ۳\n"
        "🔎 شکار — سطح ۴ و هر ۳۰ دقیقه\n"
        "🏭 کارخانه — سطح ۵\n"
        "👥 برده — سطح ۶\n"
        "📦 قاچاق — سطح ۷\n\n"
        "⚠️ بازی فقط در گروه فعال انجام می‌شود.\n"
        "در PV هیچ‌یک از قابلیت‌های بازی اجرا نمی‌شود."
    )


def install_text():
    return (
        "➕ نصب ربات انسانی در گروه\n\n"
        "برای راه‌اندازی:\n"
        "۱. ربات را به گروه اضافه کنید.\n"
        "۲. ربات را مدیر کامل کنید.\n"
        "۳. داخل گروه عبارت «نصب» را ارسال کنید.\n\n"
        "پس از نصب، بازی فقط در همان گروه فعال می‌شود."
    )


def transfer_help():
    return (
        "💸 انتقال انسانی\n\n"
        "برای انتقال:\n"
        "۱. روی پیام کاربر مقصد Reply کنید.\n"
        "۲. عبارت «انتقال انسانی» را ارسال کنید.\n"
        "۳. مقدار را بفرستید.\n\n"
        "انتقال فقط داخل گروه فعال انجام می‌شود."
    )


def set_pending(uid, action):
    db_exec(
        """INSERT INTO settings(key,value) VALUES(?,?)
           ON CONFLICT(key) DO UPDATE SET value=excluded.value""",
        (f"pending:{uid}", action)
    )


def get_pending(uid):
    row = db_one("SELECT value FROM settings WHERE key=?", (f"pending:{uid}",))
    return row["value"] if row else ""


def clear_pending(uid):
    db_exec("DELETE FROM settings WHERE key=?", (f"pending:{uid}",))



def back_private(cb):
    answer_callback(cb, "بازگشت به منوی اصلی")
    edit_message(cb, private_text(), main_menu())

def add_group(cb):
    answer_callback(cb, "راهنمای افزودن به گروه نمایش داده شد.")
    edit_message(
        cb,
        """➕ راه‌اندازی ربات انسانی

برای فعال کردن بازی در گروه:

① یک گروه با حداقل ۱۰۰ عضو داشته باشید.
② ربات را به گروه اضافه کنید.
③ ربات را مدیر گروه کنید و دسترسی‌های لازم را بدهید.
④ داخل همان گروه عبارت «نصب» را ارسال کنید.

✅ پس از ارسال «نصب»، ربات برای گروه فعال می‌شود.

⚠️ بازی و امکانات ربات فقط داخل گروه قابل استفاده است و در PV بازی امکان‌پذیر نیست.""",
        kb([[btn("↩️ برگشت", "back_private")]])
    )

def handle_text(update, msg):
    chat = msg.get("chat", {})
    user = msg.get("from", {})
    if not user.get("id"):
        return
    uid = int(user["id"])
    ensure_user(uid, user.get("first_name",""), user.get("username",""))

    # PV: هیچ بازی، ثبت عضو گروه یا قابلیت بازی اجرا نمی‌شود.
    if chat.get("type") not in GROUP_TYPES:
        # فقط پیام معرفی برای start.
        text = (msg.get("text") or "").strip().lower()
        if text in ("/start", "start"):
            send_message(
                chat["id"],
                "🤖 سلام ، من ربات انسانی هستم ؛\n"
                "یک بازی و سرگرمی جذاب برای گروه شما 🎮\n\n"
                "من رو به گروهتون اضافه کنید و با اعضای گروه وارد دنیای ربات انسانی بشید ؛ "
                "بازی کنید ، رقابت کنید و سرگرم بشید! 🚀",
                private_menu()
            )
        # عمداً هیچ دستور بازی در PV پاسخ داده نمی‌شود.
        return

    # از اینجا به بعد فقط گروه.
    cid = int(chat["id"])
    ensure_group(chat)
    record_member(cid, user)

    text = (msg.get("text") or "").strip()
    low = text.lower()

    if low == "نصب":
        db_exec(
            "UPDATE groups SET active=1,installed_at=?,last_seen=? WHERE chat_id=?",
            (now_ts(), now_ts(), cid)
        )
        send_message(
            cid,
            "✅ ربات انسانی با موفقیت در این گروه فعال شد!\n\n"
            "🎮 بازی از همین حالا در گروه قابل استفاده است.\n"
            "📖 برای مشاهده قابلیت‌ها: راهنما",
            game_menu()
        )
        return

    if not group_is_active(cid):
        if low in ("راهنما", "/help"):
            send_message(cid, "⚠️ این گروه هنوز نصب نشده است.\nعبارت «نصب» را ارسال کنید.")
        return

    # Pending actions
    pending = get_pending(uid)
    if pending.startswith("deposit:"):
        try:
            amount = int(text.replace(",", "").replace("٬",""))
            if amount <= 0:
                raise ValueError
            u = db_one("SELECT points,level FROM users WHERE user_id=?", (uid,))
            if int(u["level"]) < 2:
                raise ValueError
            if int(u["points"]) < amount:
                send_message(cid, "❌ انسان پوینت کافی ندارید.")
            else:
                db_exec(
                    "UPDATE users SET points=points-?,bank=bank+? WHERE user_id=?",
                    (amount, amount, uid)
                )
                send_message(cid, f"✅ {fa_num(amount)} انسان پوینت به بانک سپرده شد.", bank_menu())
            clear_pending(uid)
        except Exception:
            send_message(cid, "❌ مقدار نامعتبر است. فقط یک عدد مثبت بفرستید.")
        return

    if pending.startswith("withdraw:"):
        try:
            amount = int(text.replace(",", "").replace("٬",""))
            if amount <= 0:
                raise ValueError
            apply_bank_interest(uid)
            u = db_one("SELECT bank FROM users WHERE user_id=?", (uid,))
            if int(u["bank"]) < amount:
                send_message(cid, "❌ موجودی بانک کافی نیست.")
            else:
                db_exec(
                    "UPDATE users SET bank=bank-?,points=points+? WHERE user_id=?",
                    (amount, amount, uid)
                )
                send_message(cid, f"✅ {fa_num(amount)} از بانک برداشت شد.", bank_menu())
            clear_pending(uid)
        except Exception:
            send_message(cid, "❌ مقدار نامعتبر است. فقط یک عدد مثبت بفرستید.")
        return

    if pending == "casino_create":
        try:
            parts = text.replace("،", " ").split()
            if len(parts) != 2:
                raise ValueError
            entry, players = int(parts[0]), int(parts[1])
            if entry <= 0 or not 2 <= players <= 6:
                raise ValueError
            u = db_one("SELECT points FROM users WHERE user_id=?", (uid,))
            if int(u["points"]) < entry:
                send_message(cid, "❌ انسان پوینت کافی ندارید.")
                clear_pending(uid)
                return
            room = db_exec(
                """INSERT INTO casino_rooms
                   (creator_id,entry,max_players,created_at)
                   VALUES(?,?,?,?)""",
                (uid, entry, players, now_ts())
            )
            db_exec(
                "UPDATE users SET points=points-? WHERE user_id=?",
                (entry, uid)
            )
            db_exec(
                "INSERT INTO casino_players(room_id,user_id) VALUES(?,?)",
                (room, uid)
            )
            send_message(
                cid,
                "🎰 کازینو ساخته شد!\n\n"
                f"ورودی: {fa_num(entry)}\n"
                f"تعداد بازیکن: {fa_num(players)}\n"
                f"شانس هر نفر: {fa_num(100//players)}٪\n"
                f"جایزه نهایی: {fa_num(entry*players)}",
                kb([[btn("🎰 پیوستن", f"casino_join:{room}")],
                    [btn("↩️ برگشت", "back")]])
            )
        except Exception:
            send_message(cid, "فرمت درست: «مبلغ تعدادبازیکن»\nمثال: ۱۰۰ ۴")
        clear_pending(uid)
        return

    if pending == "slave_rename":
        name = text[:30].strip()
        if not name:
            send_message(cid, "❌ نام نامعتبر است.")
        else:
            db_exec("UPDATE slaves SET name=? WHERE user_id=?", (name, uid))
            send_message(cid, "✅ نام تغییر کرد.", slave_menu())
        clear_pending(uid)
        return

    if pending == "transfer_amount":
        try:
            amount = int(text.replace(",", "").replace("٬",""))
            target = db_one(
                "SELECT value FROM settings WHERE key=?",
                (f"transfer_target:{uid}",)
            )
            if not target or amount <= 0:
                raise ValueError
            tid = int(target["value"])
            u = db_one("SELECT points FROM users WHERE user_id=?", (uid,))
            if int(u["points"]) < amount:
                send_message(cid, "❌ موجودی کافی نیست.")
            else:
                db_exec("UPDATE users SET points=points-? WHERE user_id=?", (amount, uid))
                db_exec("UPDATE users SET points=points+? WHERE user_id=?", (amount, tid))
                db_exec(
                    "INSERT INTO transfers(sender_id,receiver_id,amount,created_at) VALUES(?,?,?,?)",
                    (uid, tid, amount, now_ts())
                )
                send_message(cid, f"✅ انتقال {fa_num(amount)} انسان پوینت انجام شد.")
            clear_pending(uid)
            db_exec("DELETE FROM settings WHERE key=?", (f"transfer_target:{uid}",))
        except Exception:
            send_message(cid, "❌ فقط مقدار عددی مثبت ارسال کنید.")
        return

    # Normal commands
    commands = {
        "پروفایل": lambda: (profile_text(uid), None),
        "انسان": lambda: (do_human(uid), None),
        "بانک انسانی": lambda: (bank_text(uid), bank_menu()),
        "کازینو انسانی": lambda: casino_text(uid),
        "شکار انسانی": lambda: hunt_start(uid),
        "برده انسانی": lambda: (slave_start(uid), slave_menu()),
        "قاچاق انسانی": lambda: contraband_text(uid),
        "کارخانه انسانی": lambda: (factory_start(uid), factory_menu()),
        "راهنما": lambda: (help_text(), game_menu()),
        "انتقال انسانی": lambda: (transfer_help(), None),
    }

    if text in commands:
        try:
            result = commands[text]()
            if isinstance(result, tuple):
                response, keyboard = result
            else:
                response, keyboard = result, None
            if isinstance(response, tuple):
                response, keyboard = response
            send_message(cid, response, keyboard, msg.get("message_id"))
        except PermissionError as e:
            send_message(cid, f"🔒 {e}")
        except Exception:
            logging.exception("command error")
            send_message(cid, "❌ خطایی در اجرای دستور رخ داد.")
        return

    # انتقال با reply
    if text == "انتقال انسانی" and msg.get("reply_to_message"):
        target = msg["reply_to_message"].get("from")
        if target and target.get("id") and int(target["id"]) != uid:
            tid = int(target["id"])
            ensure_user(tid, target.get("first_name",""), target.get("username",""))
            db_exec(
                """INSERT INTO settings(key,value) VALUES(?,?)
                   ON CONFLICT(key) DO UPDATE SET value=excluded.value""",
                (f"transfer_target:{uid}", str(tid))
            )
            set_pending(uid, "transfer_amount")
            send_message(cid, "💸 مقدار انسان پوینت برای انتقال را ارسال کنید.")
        return
def callback_handler(update):
    cq = update.get("callback_query", {})
    if not cq:
        return
    uid = int(cq.get("from", {}).get("id", 0))
    msg = cq.get("message", {})
    chat = msg.get("chat", {})
    cid = int(chat.get("id", 0))
    mid = int(msg.get("message_id", 0))
    data = cq.get("data", "")

    ensure_user(
        uid,
        cq.get("from", {}).get("first_name",""),
        cq.get("from", {}).get("username","")
    )

    # هر callback بازی فقط اگر در گروه فعال باشد.
    if chat.get("type") not in GROUP_TYPES:
        answer_callback(cq.get("id"), "بازی فقط داخل گروه قابل استفاده است.")
        return
    ensure_group(chat)
    record_member(cid, cq.get("from", {}))
    if not group_is_active(cid):
        answer_callback(cq.get("id"), "ابتدا «نصب» را در گروه ارسال کنید.")
        return

    if data == "noop_groups":
        answer_callback(cq.get("id"), f"گروه‌های فعال: {fa_num(count_groups())}")
        return
    if data == "noop_users":
        answer_callback(cq.get("id"), f"کاربران ثبت‌شده: {fa_num(count_users())}")
        return

    if data == "add_group":
        answer_callback(cb, "راهنمای افزودن به گروه نمایش داده شد.")
        edit_message(cb, '➕ راه\u200cاندازی ربات انسانی\n\nبرای فعال کردن بازی در گروه:\n\n① یک گروه با حداقل ۱۰۰ عضو داشته باشید.\n② ربات را به گروه اضافه کنید.\n③ ربات را مدیر گروه کنید و دسترسی\u200cهای لازم را بدهید.\n④ داخل همان گروه عبارت «نصب» را ارسال کنید.\n\n✅ پس از ارسال «نصب»، ربات برای گروه فعال می\u200cشود.\n\n⚠️ بازی و امکانات ربات فقط داخل گروه قابل استفاده است و در PV بازی امکان\u200cپذیر نیست.', kb([[btn("↩️ برگشت", "back_private")]]))
    try:
        if data == "profile":
            edit_message(cid, mid, profile_text(uid), kb([[btn("↩️ برگشت", "back")]]))
        elif data == "human":
            edit_message(cid, mid, do_human(uid), game_menu())
        elif data == "bank":
            edit_message(cid, mid, bank_text(uid), bank_menu())
        elif data == "bank_deposit":
            set_pending(uid, "deposit:await")
            answer_callback(cq.get("id"), "مقدار را در گروه ارسال کنید.")
            edit_message(cid, mid, "➕ مقدار سپرده را همین‌جا به صورت پیام ارسال کنید.", kb([[btn("↩️ برگشت", "back")]]))
        elif data == "bank_withdraw":
            set_pending(uid, "withdraw:await")
            answer_callback(cq.get("id"), "مقدار را در گروه ارسال کنید.")
            edit_message(cid, mid, "➖ مقدار برداشت را همین‌جا به صورت پیام ارسال کنید.", kb([[btn("↩️ برگشت", "back")]]))
        elif data == "casino":
            text, keyboard = casino_text(uid)
            edit_message(cid, mid, text, keyboard)
        elif data == "casino_create":
            set_pending(uid, "casino_create")
            edit_message(cid, mid, "🎰 برای ساخت کازینو دو عدد ارسال کنید:\n\nمبلغ ورودی + تعداد بازیکن\nمثال: ۱۰۰ ۴", kb([[btn("↩️ برگشت", "back")]]))
        elif data.startswith("casino_join:"):
            room = int(data.split(":")[1])
            join_casino(cid, uid, room)
        elif data == "hunt":
            text, keyboard = hunt_start(uid)
            edit_message(cid, mid, text, keyboard)
        elif data.startswith("hunt:"):
            _, hid, pos = data.split(":")
            text, keyboard = hunt_pick(uid, int(hid), int(pos))
            edit_message(cid, mid, text, keyboard)
        elif data == "slave":
            edit_message(cid, mid, slave_start(uid), slave_menu())
        elif data == "slave_upgrade":
            slave_upgrade(cid, mid, uid)
        elif data == "slave_claim":
            slave_claim(cid, mid, uid)
        elif data == "slave_rename":
            set_pending(uid, "slave_rename")
            edit_message(cid, mid, "✏️ نام جدید را به صورت پیام ارسال کنید.", kb([[btn("↩️ برگشت", "back")]]))
        elif data == "factory":
            edit_message(cid, mid, factory_start(uid), factory_menu())
        elif data == "factory_claim":
            factory_claim(cid, mid, uid)
        elif data == "factory_upgrade":
            factory_upgrade(cid, mid, uid)
        elif data == "contraband":
            text, keyboard = contraband_text(uid)
            edit_message(cid, mid, text, keyboard)
        elif data == "transfer_help":
            edit_message(cid, mid, transfer_help(), kb([[btn("↩️ برگشت", "back")]]))
        elif data == "help":
            edit_message(cid, mid, help_text(), game_menu())
        elif data == "back":
            clear_pending(uid)
            edit_message(cid, mid, group_menu_text(), game_menu())
        else:
            answer_callback(cq.get("id"), "گزینه نامعتبر است.")
    except PermissionError as e:
        edit_message(cid, mid, f"🔒 {e}", game_menu())
    except Exception:
        logging.exception("callback error")
        edit_message(cid, mid, "❌ خطایی رخ داد.", game_menu())


def join_casino(cid, uid, room):
    with user_lock(uid):
        r = db_one("SELECT * FROM casino_rooms WHERE id=?", (room,))
        if not r or r["status"] != "open":
            send_message(cid, "❌ این کازینو بسته شده است.")
            return
        already = db_one(
            "SELECT 1 FROM casino_players WHERE room_id=? AND user_id=?",
            (room, uid)
        )
        if already:
            send_message(cid, "شما قبلاً وارد شده‌اید.")
            return
        u = db_one("SELECT points FROM users WHERE user_id=?", (uid,))
        if int(u["points"]) < int(r["entry"]):
            send_message(cid, "❌ انسان پوینت کافی ندارید.")
            return
        db_exec("UPDATE users SET points=points-? WHERE user_id=?", (int(r["entry"]), uid))
        db_exec("INSERT INTO casino_players(room_id,user_id) VALUES(?,?)", (room, uid))
        players = db_all("SELECT user_id FROM casino_players WHERE room_id=?", (room,))
        if len(players) < int(r["max_players"]):
            send_message(cid, f"🎰 وارد کازینو شدی.\nبازیکنان: {fa_num(len(players))}/{fa_num(r['max_players'])}")
            return
        winner = random.choice(players)
        prize = int(r["entry"]) * int(r["max_players"])
        db_exec("UPDATE users SET points=points+? WHERE user_id=?", (prize, int(winner["user_id"])))
        db_exec("UPDATE casino_rooms SET status='closed' WHERE id=?", (room,))
        send_message(
            cid,
            "🎰 کازینو تمام شد!\n\n"
            f"💰 جایزه: {fa_num(prize)} انسان پوینت\n"
            "🏆 برنده به صورت تصادفی انتخاب شد."
        )


def slave_upgrade(cid, mid, uid):
    if not level_required(uid, 6):
        edit_message(cid, mid, "🔒 برده انسانی از سطح ۶ باز می‌شود.", slave_menu())
        return
    slave_sync(uid)
    s = db_one("SELECT * FROM slaves WHERE user_id=?", (uid,))
    lvl = int(s["level"])
    if lvl >= 20:
        edit_message(cid, mid, "🏆 برده انسانی در بالاترین سطح است.", slave_menu())
        return
    cost = SLAVE_COST[lvl]
    u = db_one("SELECT points FROM users WHERE user_id=?", (uid,))
    if int(u["points"]) < cost:
        edit_message(cid, mid, f"❌ برای ارتقا {fa_num(cost)} انسان پوینت نیاز است.", slave_menu())
        return
    db_exec("UPDATE users SET points=points-? WHERE user_id=?", (cost, uid))
    db_exec("UPDATE slaves SET level=level+1,last_tick=? WHERE user_id=?", (now_ts(), uid))
    edit_message(cid, mid, slave_start(uid), slave_menu())


def slave_claim(cid, mid, uid):
    slave_sync(uid)
    s = db_one("SELECT * FROM slaves WHERE user_id=?", (uid,))
    amount = int(float(s["stored"]))
    if amount <= 0:
        edit_message(cid, mid, "⏳ هنوز پوینتی برای دریافت آماده نیست.", slave_menu())
        return
    db_exec("UPDATE users SET points=points+? WHERE user_id=?", (amount, uid))
    db_exec("UPDATE slaves SET stored=stored-? WHERE user_id=?", (amount, uid))
    edit_message(cid, mid, f"✅ {fa_num(amount)} انسان پوینت دریافت شد.", slave_menu())


def factory_claim(cid, mid, uid):
    if not level_required(uid, 5):
        edit_message(cid, mid, "🔒 کارخانه انسانی از سطح ۵ باز می‌شود.", factory_menu())
        return
    factory_start(uid)
    f = db_one("SELECT ready FROM factories WHERE user_id=?", (uid,))
    amount = int(f["ready"])
    if amount <= 0:
        edit_message(cid, mid, factory_start(uid), factory_menu())
        return
    db_exec("UPDATE users SET slaves=slaves+? WHERE user_id=?", (amount, uid))
    db_exec("UPDATE factories SET ready=0 WHERE user_id=?", (uid,))
    edit_message(
        cid, mid,
        f"✅ {fa_num(amount)} واحد انسانی مجازی دریافت شد.\n\n" + factory_start(uid),
        factory_menu()
    )


def factory_upgrade(cid, mid, uid):
    if not level_required(uid, 5):
        edit_message(cid, mid, "🔒 کارخانه انسانی از سطح ۵ باز می‌شود.", factory_menu())
        return
    factory_start(uid)
    f = db_one("SELECT level FROM factories WHERE user_id=?", (uid,))
    lvl = int(f["level"])
    if lvl >= 7:
        edit_message(cid, mid, "🏆 کارخانه در بالاترین سطح است.", factory_menu())
        return
    cost = FACTORY_COST[lvl]
    u = db_one("SELECT points FROM users WHERE user_id=?", (uid,))
    if int(u["points"]) < cost:
        edit_message(cid, mid, f"❌ {fa_num(cost)} انسان پوینت نیاز است.", factory_menu())
        return
    db_exec("UPDATE users SET points=points-? WHERE user_id=?", (cost, uid))
    db_exec("UPDATE factories SET level=level+1 WHERE user_id=?", (uid,))
    edit_message(cid, mid, factory_start(uid), factory_menu())


def process_update(update):
    try:
        if "message" in update:
            handle_text(update, update["message"])
        elif "callback_query" in update:
            callback_handler(update)
    except Exception:
        logging.error("UPDATE ERROR\n%s", traceback.format_exc())


def main():
    if not TOKEN.strip():
        print("TOKEN خالی است.")
        print('در ابتدای فایل مقدار زیر را وارد کنید:')
        print('TOKEN = "توکن_ربات_شما"')
        return

    init_db()
    migrate_hunt_targets()
    logging.info("HUMAN BOT started.")

    offset = 0
    while True:
        try:
            result = api("getUpdates", {
                "offset": offset,
                "timeout": POLL_TIMEOUT
            })
            updates = result.get("result", []) if isinstance(result, dict) else []
            for upd in updates:
                offset = int(upd.get("update_id", offset)) + 1
                process_update(upd)
        except KeyboardInterrupt:
            break
        except Exception as e:
            logging.error("POLL ERROR: %s", e)
            time.sleep(3)


if __name__ == "__main__":
    main()
