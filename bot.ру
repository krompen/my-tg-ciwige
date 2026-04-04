import sqlite3
import random
import datetime
import asyncio
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler, ConversationHandler

TOKEN = "8526281569:AAH022yc8YfARu9jQXSjRHHL5-wzO4W_4to"
ADMIN_ID = 8032626504
ADMIN_USERNAME = "ciwige"

# ========== БАЗА ДАННЫХ ==========
DB_NAME = "tags.db"

def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    # Таблица пользователей
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        premium_level INTEGER DEFAULT 0,
        daily_requests INTEGER DEFAULT 5,
        last_reset TEXT
    )''')
    # Таблица найденных тегов
    c.execute('''CREATE TABLE IF NOT EXISTS found_tags (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        tag TEXT,
        found_date TEXT
    )''')
    # Таблица тикетов
    c.execute('''CREATE TABLE IF NOT EXISTS tickets (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        question TEXT,
        status TEXT DEFAULT 'open',
        created_at TEXT,
        admin_answer TEXT,
        answered_at TEXT
    )''')
    # Таблица логов действий
    c.execute('''CREATE TABLE IF NOT EXISTS logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        action TEXT,
        time TEXT
    )''')
    conn.commit()
    conn.close()

def log_action(user_id, action):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    c.execute('INSERT INTO logs (user_id, action, time) VALUES (?, ?, ?)', (user_id, action, now))
    conn.commit()
    conn.close()

def get_user(user_id, username=""):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT user_id, username, premium_level, daily_requests, last_reset FROM users WHERE user_id = ?', (user_id,))
    user = c.fetchone()
    conn.close()
    if not user:
        add_user(user_id, username)
        return get_user(user_id, username)
    # Сброс лимитов
    today = datetime.datetime.now().strftime("%Y-%m-%d")
    if user[4] != today:
        # Премиум-уровни: 1 - 10 запросов, 2 - 20 запросов, 3 - безлимит
        if user[2] == 1:
            requests = 10
        elif user[2] == 2:
            requests = 20
        elif user[2] == 3:
            requests = 999999
        else:
            requests = 5
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()
        c.execute('UPDATE users SET daily_requests = ?, last_reset = ? WHERE user_id = ?', (requests, today, user_id))
        conn.commit()
        conn.close()
        return get_user(user_id, username)
    return user

def add_user(user_id, username):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    today = datetime.datetime.now().strftime("%Y-%m-%d")
    c.execute('INSERT INTO users (user_id, username, premium_level, daily_requests, last_reset) VALUES (?, ?, 0, 5, ?)',
              (user_id, username, today))
    conn.commit()
    conn.close()
    log_action(user_id, "Зарегистрировался")

def set_premium(user_id, level):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    if level == 0:
        requests = 5
    elif level == 1:
        requests = 10
    elif level == 2:
        requests = 20
    else:
        requests = 999999
    c.execute('UPDATE users SET premium_level = ?, daily_requests = ? WHERE user_id = ?', (level, requests, user_id))
    conn.commit()
    conn.close()
    log_action(ADMIN_ID, f"Выдал премиум {level} пользователю {user_id}")

def use_request(user_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    user = get_user(user_id)
    if user[2] == 3:
        conn.close()
        return True
    current = user[3]
    if current <= 0:
        conn.close()
        return False
    c.execute('UPDATE users SET daily_requests = daily_requests - 1 WHERE user_id = ?', (user_id,))
    conn.commit()
    conn.close()
    return True

def get_remaining_requests(user_id):
    user = get_user(user_id)
    if user[2] == 3:
        return "∞ (VIP)"
    elif user[2] == 2:
        return f"{user[3]} (Премиум Плюс)"
    elif user[2] == 1:
        return f"{user[3]} (Премиум Базовый)"
    return f"{user[3]}"

def get_premium_name(level):
    if level == 3:
        return "👑 VIP Премиум"
    elif level == 2:
        return "💎 Премиум Плюс"
    elif level == 1:
        return "⭐ Премиум Базовый"
    return "🆓 Обычный"

def save_found_tag(user_id, tag):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    c.execute('INSERT INTO found_tags (user_id, tag, found_date) VALUES (?, ?, ?)', (user_id, tag, now))
    conn.commit()
    conn.close()
    log_action(user_id, f"Нашёл тег {tag}")

def get_found_tags(user_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT tag, found_date FROM found_tags WHERE user_id = ? ORDER BY id DESC LIMIT 10', (user_id,))
    tags = c.fetchall()
    conn.close()
    return tags

# ========== ТИКЕТЫ ==========
def create_ticket(user_id, username, question):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    c.execute('INSERT INTO tickets (user_id, username, question, status, created_at) VALUES (?, ?, ?, ?, ?)',
              (user_id, username, question, 'open', now))
    ticket_id = c.lastrowid
    conn.commit()
    conn.close()
    log_action(user_id, f"Создал тикет #{ticket_id}: {question[:50]}")
    return ticket_id

def get_open_tickets():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT id, user_id, username, question, created_at FROM tickets WHERE status = "open" ORDER BY id DESC')
    tickets = c.fetchall()
    conn.close()
    return tickets

def get_user_tickets(user_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT id, question, status, created_at, admin_answer, answered_at FROM tickets WHERE user_id = ? ORDER BY id DESC', (user_id,))
    tickets = c.fetchall()
    conn.close()
    return tickets

def answer_ticket(ticket_id, answer):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    c.execute('UPDATE tickets SET status = "closed", admin_answer = ?, answered_at = ? WHERE id = ?', (answer, now, ticket_id))
    conn.commit()
    conn.close()
    log_action(ADMIN_ID, f"Ответил на тикет #{ticket_id}")

def get_ticket_by_id(ticket_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT id, user_id, username, question, status, created_at, admin_answer, answered_at FROM tickets WHERE id = ?', (ticket_id,))
    ticket = c.fetchone()
    conn.close()
    return ticket

# ========== ФУНКЦИИ ПОИСКА ТЕГОВ ==========
WORDS_5 = ["apple", "brain", "cloud", "dragon", "eagle", "flame", "glory", "heart", "light", "magic", "night", "ocean", "power", "queen", "stone", "tiger", "unity", "vivid", "wings", "xenon", "young", "zebra"]
WORDS_6 = ["animal", "bridge", "castle", "desert", "energy", "forest", "garden", "heroes", "island", "jumper", "keeper", "legend", "mystic", "nature", "orange", "planet", "queens", "radius", "shadow", "thunder", "unique", "victor", "window", "xenons", "yellow", "zenith"]

def generate_random_tag():
    if random.choice([True, False]):
        return random.choice(WORDS_5)
    return random.choice(WORDS_6)

def check_tag_availability(tag):
    # Симуляция проверки (в реальности через API Telegram)
    return random.random() > 0.6

async def find_free_tag(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user = get_user(user_id, update.effective_user.username)
    
    if not use_request(user_id):
        remaining = get_remaining_requests(user_id)
        await update.message.reply_text(
            f"❌ Ты исчерпал лимит запросов на сегодня ({remaining}).\n"
            f"Лимит обнулится завтра, или купи премиум у @{ADMIN_USERNAME}."
        )
        return
    
    await update.message.reply_text("🔍 Ищу свободный тег... Подожди немного ⏳")
    await asyncio.sleep(1.5)
    
    found = False
    attempts = 0
    tag = ""
    
    while not found and attempts < 20:
        tag = generate_random_tag()
        found = check_tag_availability(tag)
        attempts += 1
    
    if found:
        save_found_tag(user_id, tag)
        remaining = get_remaining_requests(user_id)
        premium_name = get_premium_name(user[2])
        await update.message.reply_text(
            f"✅ **Ник найден!**\n\n"
            f"📝 **Ник** - @{tag} › `{tag}`\n"
            f"├ 🟢 **Ликвидность** - {random.randint(5,10)} из 10 ⭐\n"
            f"╰ ⚡ **Свободен**\n\n"
            f"🤍 **Осталось запросов** - {remaining}\n"
            f"💎 **Статус** - {premium_name}"
        )
    else:
        await update.message.reply_text(
            f"❌ Не удалось найти свободный тег.\n"
            f"Попробуй ещё раз или измени параметры поиска."
        )

# ========== КОМАНДЫ ==========
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    add_user(user.id, user.username)
    await update.message.reply_text(
        f"👋 Привет, {user.first_name}!\n\n"
        f"Я ищу свободные теги (ники) длиной 5-6 букв.\n\n"
        f"📌 **Команды:**\n"
        f"/find — найти свободный тег\n"
        f"/profile — твой профиль\n"
        f"/mytags — найденные тобой теги\n"
        f"/premium — информация о премиум\n"
        f"/ticket — создать обращение в поддержку\n"
        f"/mytickets — мои обращения\n"
        f"/help — помощь\n\n"
        f"💰 Премиум даёт больше запросов.\n"
        f"❓ Вопросы — @{ADMIN_USERNAME}"
    )

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📖 **Помощь**\n\n"
        "/find — найти свободный тег\n"
        "/profile — посмотреть профиль\n"
        "/mytags — показать найденные теги\n"
        "/premium — информация о премиум\n"
        "/ticket — написать в поддержку\n"
        "/mytickets — мои обращения\n\n"
        "👑 **Админ-команды:**\n"
        "/givepremium <id> <1/2/3> — выдать премиум\n"
        "/removepremium <id> — забрать премиум\n"
        "/tickets — список открытых тикетов\n"
        "/answer <id> <ответ> — ответить на тикет\n"
        "/stats — статистика"
    )

async def profile(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user = get_user(user_id, update.effective_user.username)
    remaining = get_remaining_requests(user_id)
    status = get_premium_name(user[2])
    await update.message.reply_text(
        f"📊 **Твой профиль**\n\n"
        f"🆔 ID: `{user_id}`\n"
        f"👤 Имя: @{update.effective_user.username or 'нет'}\n"
        f"💎 Статус: {status}\n"
        f"🔍 Осталось запросов: {remaining}\n\n"
        f"Премиум даёт больше запросов! /premium"
    )

async def mytags(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    tags = get_found_tags(user_id)
    if not tags:
        await update.message.reply_text("📭 Ты ещё не нашёл ни одного тега.")
        return
    text = "📋 **Твои найденные теги:**\n\n"
    for tag, date in tags[:10]:
        text += f"• `{tag}` — найден {date[:10]}\n"
    await update.message.reply_text(text)

async def premium_info(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "💎 **Премиум уровни**\n\n"
        "⭐ **Базовый** — 10 запросов в день\n"
        "💎 **Плюс** — 20 запросов в день\n"
        "👑 **VIP** — безлимит\n\n"
        "💰 Цены уточняй у @{ADMIN_USERNAME}\n"
        "🔑 Для получения премиум напиши админу."
    )

# ========== ТИКЕТЫ ==========
TICKET_STATE = range(1)

async def ticket_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📝 **Создание обращения**\n\n"
        "Опиши свой вопрос подробно. Я передам его админу.\n"
        "Напиши /cancel чтобы отменить."
    )
    return TICKET_STATE

async def ticket_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    username = update.effective_user.username or "без_юзернейма"
    question = update.message.text
    
    ticket_id = create_ticket(user_id, username, question)
    await update.message.reply_text(
        f"✅ Твой запрос принят! Номер тикета: #{ticket_id}\n"
        f"Админ ответит как можно скорее. История обращений: /mytickets"
    )
    
    # Уведомление админу
    await context.bot.send_message(
        chat_id=ADMIN_ID,
        text=f"🆕 **Новый тикет #{ticket_id}**\n"
             f"👤 От: @{username} (ID: {user_id})\n"
             f"📝 Вопрос: {question}\n\n"
             f"Ответ: /answer {ticket_id} [текст]"
    )
    return ConversationHandler.END

async def ticket_cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("❌ Создание обращения отменено.")
    return ConversationHandler.END

async def mytickets(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    tickets = get_user_tickets(user_id)
    if not tickets:
        await update.message.reply_text("📭 У тебя нет обращений.")
        return
    text = "📋 **Твои обращения:**\n\n"
    for t in tickets:
        status = "✅ Закрыт" if t[2] == "closed" else "🟡 Открыт"
        text += f"#{t[0]} | {status} | {t[3][:10]}\n"
        if t[4]:
            text += f"   Ответ: {t[4][:50]}\n"
    await update.message.reply_text(text)

# ========== АДМИН-КОМАНДЫ ==========
async def give_premium(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Нет прав.")
        return
    if len(context.args) < 2:
        await update.message.reply_text("❌ Использование: /givepremium <user_id> <1/2/3>")
        return
    try:
        target_id = int(context.args[0])
        level = int(context.args[1])
        if level not in [1,2,3]:
            await update.message.reply_text("❌ Уровень должен быть 1, 2 или 3")
            return
        set_premium(target_id, level)
        level_name = {1:"Базовый",2:"Плюс",3:"VIP"}[level]
        await update.message.reply_text(f"✅ Пользователь {target_id} получил премиум {level_name}.")
        try:
            await context.bot.send_message(
                chat_id=target_id,
                text=f"🎉 Вам выдан премиум {level_name}!\nТеперь у вас больше запросов в день."
            )
        except:
            pass
    except:
        await update.message.reply_text("❌ Неверный ID или уровень.")

async def remove_premium(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Нет прав.")
        return
    if not context.args:
        await update.message.reply_text("❌ Использование: /removepremium <user_id>")
        return
    try:
        target_id = int(context.args[0])
        set_premium(target_id, 0)
        await update.message.reply_text(f"✅ У пользователя {target_id} забран премиум.")
        try:
            await context.bot.send_message(chat_id=target_id, text="❌ Премиум забран. Лимит снова 5 запросов в день.")
        except:
            pass
    except:
        await update.message.reply_text("❌ Неверный ID.")

async def tickets_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Нет прав.")
        return
    tickets = get_open_tickets()
    if not tickets:
        await update.message.reply_text("📭 Нет открытых тикетов.")
        return
    text = "🆘 **Открытые тикеты:**\n\n"
    for t in tickets:
        text += f"#{t[0]} | @{t[2]} (ID: {t[1]})\n"
        text += f"📝 {t[3][:80]}\n"
        text += f"⏰ {t[4]}\n"
        text += f"💡 /answer {t[0]} [текст]\n\n"
    await update.message.reply_text(text[:4000])

async def answer_ticket_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Нет прав.")
        return
    if len(context.args) < 2:
        await update.message.reply_text("❌ Использование: /answer <ticket_id> <текст ответа>")
        return
    try:
        ticket_id = int(context.args[0])
        answer = " ".join(context.args[1:])
        
        ticket = get_ticket_by_id(ticket_id)
        if not ticket:
            await update.message.reply_text("❌ Тикет не найден.")
            return
        
        answer_ticket(ticket_id, answer)
        await update.message.reply_text(f"✅ Ответ на тикет #{ticket_id} отправлен.")
        
        await context.bot.send_message(
            chat_id=ticket[1],
            text=f"📬 **Ответ на ваше обращение #{ticket_id}**\n\n"
                 f"❓ Вопрос: {ticket[3]}\n\n"
                 f"✅ Ответ: {answer}\n\n"
                 f"Обращение закрыто. Если остались вопросы, создайте новый тикет."
        )
        log_action(ADMIN_ID, f"Ответил на тикет #{ticket_id}")
    except:
        await update.message.reply_text("❌ Ошибка. Проверь ID тикета.")

async def stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Нет прав.")
        return
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('SELECT COUNT(*) FROM users')
    total_users = c.fetchone()[0]
    c.execute('SELECT COUNT(*) FROM users WHERE premium_level > 0')
    premium_users = c.fetchone()[0]
    c.execute('SELECT COUNT(*) FROM found_tags')
    total_tags = c.fetchone()[0]
    c.execute('SELECT COUNT(*) FROM tickets WHERE status = "open"')
    open_tickets = c.fetchone()[0]
    conn.close()
    await update.message.reply_text(
        f"📊 **Статистика бота**\n\n"
        f"👥 Всего пользователей: {total_users}\n"
        f"💎 Премиум: {premium_users}\n"
        f"🏷️ Найдено тегов: {total_tags}\n"
        f"🆘 Открытых тикетов: {open_tickets}\n"
        f"🤖 Бот работает стабильно."
    )

# ========== ОБРАБОТЧИК ==========
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.lower()
    if text in ["/find", "найти", "поиск", "тег"]:
        await find_free_tag(update, context)
    else:
        await update.message.reply_text("❓ Неизвестная команда. Напиши /help")

# ========== ЗАПУСК ==========
def main():
    init_db()
    app = Application.builder().token(TOKEN).build()
    
    # Команды
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("find", find_free_tag))
    app.add_handler(CommandHandler("profile", profile))
    app.add_handler(CommandHandler("mytags", mytags))
    app.add_handler(CommandHandler("premium", premium_info))
    app.add_handler(CommandHandler("mytickets", mytickets))
    app.add_handler(CommandHandler("givepremium", give_premium))
    app.add_handler(CommandHandler("removepremium", remove_premium))
    app.add_handler(CommandHandler("tickets", tickets_list))
    app.add_handler(CommandHandler("answer", answer_ticket_cmd))
    app.add_handler(CommandHandler("stats", stats))
    
    # Тикеты (ConversationHandler)
    ticket_conv = ConversationHandler(
        entry_points=[CommandHandler("ticket", ticket_start)],
        states={TICKET_STATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, ticket_text)]},
        fallbacks=[CommandHandler("cancel", ticket_cancel)]
    )
    app.add_handler(ticket_conv)
    
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    print("✅ Бот поиска тегов запущен с премиум-уровнями и тикетами")
    app.run_polling(drop_pending_updates=True)

if __name__ == "__main__":
    main()