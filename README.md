import telebot
from telebot import types
import sqlite3
import random
import time
from datetime import datetime
import threading

TOKEN = '8796002500:AAETEiSc6hH2ymgujQYpcoAY1b_0Yag5his'
bot = telebot.TeleBot(TOKEN)

conn = sqlite3.connect('durka_weed.db', check_same_thread=False)
cursor = conn.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT DEFAULT 'Unknown',
        grams REAL DEFAULT 0,
        money INTEGER DEFAULT 0,
        level INTEGER DEFAULT 1,
        is_banned INTEGER DEFAULT 0,
        grow_start_time REAL DEFAULT 0,
        is_growing INTEGER DEFAULT 0,
        total_harvested REAL DEFAULT 0,
        total_sold REAL DEFAULT 0,
        last_smoke_time REAL DEFAULT 0,
        grow_end_time REAL DEFAULT 0
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS admins (
        user_id INTEGER PRIMARY KEY
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS weed_price (
        id INTEGER PRIMARY KEY,
        price INTEGER DEFAULT 2000
    )
''')

cursor.execute("INSERT OR IGNORE INTO weed_price (id, price) VALUES (1, 2000)")
conn.commit()

ADMIN_PASSWORD = "123890"
SMOKE_COOLDOWN = 1800
GROW_TIME = 3600

def get_user(user_id):
    cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    return cursor.fetchone()

def create_user(user_id, username):
    if not get_user(user_id):
        cursor.execute("INSERT INTO users (user_id, username) VALUES (?, ?)", 
                      (user_id, str(username) if username else f"user{user_id}"))
        conn.commit()

def get_current_price():
    cursor.execute("SELECT price FROM weed_price WHERE id = 1")
    result = cursor.fetchone()
    return result[0] if result else 2000

def update_price():
    while True:
        try:
            new_price = random.randint(1500, 2800)
            cursor.execute("UPDATE weed_price SET price = ? WHERE id = 1", (new_price,))
            conn.commit()
        except:
            pass
        time.sleep(3600)

price_thread = threading.Thread(target=update_price, daemon=True)
price_thread.start()

def generate_grams():
    weights = [2, 3, 4, 5, 6, 7, 8, 9, 10]
    probabilities = [0.25, 0.22, 0.18, 0.13, 0.09, 0.06, 0.04, 0.02, 0.01]
    return random.choices(weights, weights=probabilities, k=1)[0]

def get_smoke_cooldown(user_id):
    user = get_user(user_id)
    if not user or not user[10]:
        return 0
    elapsed = time.time() - user[10]
    remaining = SMOKE_COOLDOWN - elapsed
    return max(0, int(remaining))

def is_admin(user_id):
    cursor.execute("SELECT * FROM admins WHERE user_id = ?", (user_id,))
    return cursor.fetchone() is not None

def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton("🌿 Вырастить шишку"),
        types.KeyboardButton("⏳ Статус роста"),
        types.KeyboardButton("🎒 Инвентарь"),
        types.KeyboardButton("💰 Продать боту"),
        types.KeyboardButton("💨 Курить шишку"),
        types.KeyboardButton("👤 Профиль"),
        types.KeyboardButton("🏆 Топ игроков"),
        types.KeyboardButton("📊 Курс шишек"),
        types.KeyboardButton("💱 Продать игроку")
    )
    return markup

def admin_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton("🔨 Забанить"),
        types.KeyboardButton("✅ Разбанить"),
        types.KeyboardButton("🌿 Выдать граммы"),
        types.KeyboardButton("🔄 Обнулить"),
        types.KeyboardButton("📋 Список"),
        types.KeyboardButton("🔙 Главное меню")
    )
    return markup

def check_growth():
    while True:
        try:
            conn2 = sqlite3.connect('durka_weed.db')
            cur2 = conn2.cursor()
            cur2.execute("SELECT user_id, grow_end_time FROM users WHERE is_growing = 1")
            growing = cur2.fetchall()
            
            for user in growing:
                user_id = user[0]
                end_time = user[1]
                
                if time.time() >= end_time:
                    grams = generate_grams()
                    
                    cursor.execute("UPDATE users SET grams = grams + ?, is_growing = 0, total_harvested = total_harvested + ?, grow_start_time = 0, grow_end_time = 0 WHERE user_id = ?", 
                                 (grams, grams, user_id))
                    conn.commit()
                    
                    try:
                        bot.send_message(user_id, f"🎉 ШИШКА ВЫРОСЛА!\n\n🌿 Вес: {grams}гр\n💰 По курсу: {int(grams * get_current_price()):,}₽\n\nМожно сажать новую!")
                    except:
                        pass
            
            conn2.close()
            time.sleep(1)
        except:
            time.sleep(5)

growth_thread = threading.Thread(target=check_growth, daemon=True)
growth_thread.start()

@bot.message_handler(commands=['start'])
def start(message):
    user_id = message.from_user.id
    username = message.from_user.username
    create_user(user_id, username)
    bot.send_message(
        message.chat.id,
        "🌿 *Добро пожаловать в Дурку!*\n\n🎮 Выращивай шишки, кури и богатей!\n⏳ Шишки растут ровно 1 час\n🎲 Вес: от 2 до 10 грамм\n💰 Курс обновляется каждый час",
        parse_mode="Markdown",
        reply_markup=main_menu()
    )

@bot.message_handler(commands=['admin'])
def admin_login(message):
    try:
        password = message.text.split()[1]
        if password == ADMIN_PASSWORD:
            cursor.execute("INSERT OR IGNORE INTO admins (user_id) VALUES (?)", (message.from_user.id,))
            conn.commit()
            bot.send_message(message.chat.id, "✅ Админка!", reply_markup=admin_menu())
        else:
            bot.send_message(message.chat.id, "❌ Неверный пароль!")
    except:
        bot.send_message(message.chat.id, "❌ /admin пароль")

@bot.message_handler(content_types=['text'])
def handle_text(message):
    try:
        user_id = message.from_user.id
        username = message.from_user.username
        create_user(user_id, username)
        user = get_user(user_id)
        
        if user and user[5] == 1:
            bot.send_message(message.chat.id, "⛔ Вы забанены!")
            return
        
        text = message.text
        
        if text == "🌿 Вырастить шишку":
            if user[8] == 1:
                remaining = user[11] - time.time()
                if remaining > 0:
                    end_time = datetime.fromtimestamp(user[11])
                    hours = int(remaining // 3600)
                    mins = int((remaining % 3600) // 60)
                    secs = int(remaining % 60)
                    bot.send_message(
                        message.chat.id,
                        f"⏳ *Шишка уже растёт!*\n\n"
                        f"🕐 Вырастет: *{end_time.strftime('%H:%M:%S')}*\n"
                        f"📅 Дата: *{end_time.strftime('%d.%m.%Y')}*\n"
                        f"⏰ Осталось: *{hours}ч {mins}м {secs}с*",
                        parse_mode="Markdown"
                    )
                    return
                else:
                    grams = generate_grams()
                    cursor.execute("UPDATE users SET grams = grams + ?, is_growing = 0, total_harvested = total_harvested + ?, grow_start_time = 0, grow_end_time = 0 WHERE user_id = ?", 
                                 (grams, grams, user_id))
                    conn.commit()
                    bot.send_message(message.chat.id, f"🎉 *ШИШКА ВЫРОСЛА!*\n\n🌿 Вес: *{grams}гр*\n💰 По курсу: *{int(grams * get_current_price()):,}₽*", parse_mode="Markdown")
                    return
            
            grow_end_time = time.time() + GROW_TIME
            end_time = datetime.fromtimestamp(grow_end_time)
            
            cursor.execute("UPDATE users SET grow_start_time = ?, is_growing = 1, grow_end_time = ? WHERE user_id = ?", 
                         (time.time(), grow_end_time, user_id))
            conn.commit()
            bot.send_message(
                message.chat.id,
                f"🌱 *Шишка посажена!*\n\n"
                f"🕐 Вырастет: *{end_time.strftime('%H:%M:%S')}*\n"
                f"📅 Дата: *{end_time.strftime('%d.%m.%Y')}*\n"
                f"⏳ Ровно через: *1 час*",
                parse_mode="Markdown"
            )
        
        elif text == "⏳ Статус роста":
            if user[8] == 1:
                remaining = user[11] - time.time()
                if remaining > 0:
                    end_time = datetime.fromtimestamp(user[11])
                    hours = int(remaining // 3600)
                    mins = int((remaining % 3600) // 60)
                    secs = int(remaining % 60)
                    progress = int(((GROW_TIME - remaining) / GROW_TIME) * 100)
                    bar_length = 20
                    filled = int(bar_length * progress / 100)
                    bar = "█" * filled + "░" * (bar_length - filled)
                    
                    bot.send_message(
                        message.chat.id,
                        f"⏳ *СТАТУС ВЫРАЩИВАНИЯ*\n\n"
                        f"🌱 Прогресс: [{bar}] *{progress}%*\n\n"
                        f"🕐 Вырастет: *{end_time.strftime('%H:%M:%S')}*\n"
                        f"📅 Дата: *{end_time.strftime('%d.%m.%Y')}*\n"
                        f"⏰ Осталось: *{hours}ч {mins}м {secs}с*",
                        parse_mode="Markdown"
                    )
                else:
                    bot.send_message(message.chat.id, "🎉 Шишка уже выросла! Нажми '🌿 Вырастить шишку' чтобы собрать!")
            else:
                bot.send_message(message.chat.id, "❌ У вас нет растущей шишки. Посадите новую!")
        
        elif text == "🎒 Инвентарь":
            user = get_user(user_id)
            cd = get_smoke_cooldown(user_id)
            if cd > 0:
                mins = cd // 60
                secs = cd % 60
                cd_text = f"⏳ {mins}м {secs}с"
            else:
                cd_text = "✅ Готов"
            
            bot.send_message(
                message.chat.id,
                f"🎒 *Инвентарь @{user[1]}*\n\n"
                f"🌿 Шишки: *{user[2]:.1f}гр*\n"
                f"💵 Деньги: *{user[3]:,}₽*\n"
                f"⭐ Уровень: *{user[4]}*\n"
                f"📊 Выращено: *{user[9]:.1f}гр*\n"
                f"💰 Продано: *{user[10]:.1f}гр*\n"
                f"💨 КД курения: {cd_text}",
                parse_mode="Markdown"
            )
        
        elif text == "💰 Продать боту":
            msg = bot.send_message(message.chat.id, "💰 Введите количество грамм:\nНапример: 2.5")
            bot.register_next_step_handler(msg, process_sell_bot)
        
        elif text == "💨 Курить шишку":
            cd = get_smoke_cooldown(user_id)
            if cd > 0:
                mins = cd // 60
                secs = cd % 60
                bot.send_message(message.chat.id, f"⏳ КД курения! Ждите {mins}м {secs}с")
                return
            msg = bot.send_message(message.chat.id, "💨 Введите граммы:\nНапример: 1")
            bot.register_next_step_handler(msg, process_smoke)
        
        elif text == "👤 Профиль":
            user = get_user(user_id)
            cd = get_smoke_cooldown(user_id)
            cd_text = "✅ Готов" if cd == 0 else f"⏳ {cd//60}м {cd%60}с"
            
            bot.send_message(
                message.chat.id,
                f"👤 *Профиль @{user[1]}*\n\n"
                f"🌿 Шишки: *{user[2]:.1f}гр*\n"
                f"💵 Деньги: *{user[3]:,}₽*\n"
                f"⭐ Уровень: *{user[4]}/100000*\n"
                f"📊 Прогресс: *{user[4]/1000:.1f}%*\n"
                f"🌱 Выращено: *{user[9]:.1f}гр*\n"
                f"💨 КД: {cd_text}",
                parse_mode="Markdown"
            )
        
        elif text == "🏆 Топ игроков":
            cursor.execute("SELECT username, grams, level FROM users WHERE is_banned = 0 ORDER BY grams DESC LIMIT 10")
            top = cursor.fetchall()
            txt = "🏆 *ТОП-10*\n\n"
            for i, u in enumerate(top, 1):
                medal = ["🥇", "🥈", "🥉"][i-1] if i <= 3 else f"{i}."
                txt += f"{medal} @{u[0]}: *{u[1]:.1f}гр* ({u[2]}лвл)\n"
            bot.send_message(message.chat.id, txt, parse_mode="Markdown")
        
        elif text == "📊 Курс шишек":
            price = get_current_price()
            bot.send_message(message.chat.id, f"📊 *Курс шишек*\n\n💰 1гр = *{price:,}₽*\n📈 Диапазон: 1500₽ - 2800₽\n🔄 Обновляется каждый час", parse_mode="Markdown")
        
        elif text == "💱 Продать игроку":
            msg = bot.send_message(message.chat.id, "💱 Введите: граммы цена @username\nПример: 2.5 5000 @username")
            bot.register_next_step_handler(msg, process_sell_player)
        
        elif is_admin(user_id):
            if text == "🔨 Забанить":
                msg = bot.send_message(message.chat.id, "Введите @username:")
                bot.register_next_step_handler(msg, process_ban)
            elif text == "✅ Разбанить":
                msg = bot.send_message(message.chat.id, "Введите @username:")
                bot.register_next_step_handler(msg, process_unban)
            elif text == "🌿 Выдать граммы":
                msg = bot.send_message(message.chat.id, "Введите: @username количество\nПример: @user 10")
                bot.register_next_step_handler(msg, process_give)
            elif text == "🔄 Обнулить":
                msg = bot.send_message(message.chat.id, "Введите @username:")
                bot.register_next_step_handler(msg, process_reset)
            elif text == "📋 Список":
                cursor.execute("SELECT username, grams, money, level FROM users LIMIT 20")
                users = cursor.fetchall()
                txt = "📋 *Игроки:*\n\n"
                for u in users:
                    txt += f"@{u[0]}: *{u[1]:.1f}гр* | {u[2]:,}₽ | {u[3]}лвл\n"
                bot.send_message(message.chat.id, txt, parse_mode="Markdown")
            elif text == "🔙 Главное меню":
                bot.send_message(message.chat.id, "✅ Меню", reply_markup=main_menu())
    
    except Exception as e:
        print(f"Error: {e}")

def process_sell_bot(message):
    try:
        grams = float(message.text)
        user_id = message.from_user.id
        user = get_user(user_id)
        
        if grams <= 0 or grams > user[2]:
            bot.send_message(message.chat.id, "❌ Неверное количество!")
            return
        
        price = get_current_price()
        total = int(grams * price)
        
        cursor.execute("UPDATE users SET grams = grams - ?, money = money + ?, total_sold = total_sold + ? WHERE user_id = ?", 
                      (grams, total, grams, user_id))
        conn.commit()
        
        bot.send_message(message.chat.id, f"✅ *Продано!*\n🌿 {grams}гр за *{total:,}₽*\n💰 Баланс: *{user[3] + total:,}₽*", parse_mode="Markdown")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка! Введите число")

def process_smoke(message):
    try:
        grams = float(message.text)
        user_id = message.from_user.id
        user = get_user(user_id)
        
        cd = get_smoke_cooldown(user_id)
        if cd > 0:
            mins = cd // 60
            secs = cd % 60
            bot.send_message(message.chat.id, f"⏳ КД! Ждите {mins}м {secs}с")
            return
        
        if grams <= 0 or grams > user[2]:
            bot.send_message(message.chat.id, "❌ Недостаточно шишек!")
            return
        
        new_level = min(user[4] + int(grams * 3), 100000)
        
        cursor.execute("UPDATE users SET grams = grams - ?, level = ?, last_smoke_time = ? WHERE user_id = ?", 
                      (grams, new_level, time.time(), user_id))
        conn.commit()
        
        bot.send_message(message.chat.id, f"💨 *Покурили!*\n🌿 Потрачено: *{grams}гр*\n⭐ Уровней: *+{int(grams*3)}*\n📊 Уровень: *{new_level}/100000*\n⏳ КД: 30 минут", parse_mode="Markdown")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка! Введите число")

def process_sell_player(message):
    try:
        parts = message.text.split()
        grams = float(parts[0])
        price = int(parts[1])
        target = parts[2].replace('@', '')
        
        user = get_user(message.from_user.id)
        if grams > user[2]:
            bot.send_message(message.chat.id, "❌ Мало шишек!")
            return
        
        cursor.execute("SELECT user_id FROM users WHERE username = ?", (target,))
        buyer = cursor.fetchone()
        if not buyer:
            bot.send_message(message.chat.id, "❌ Игрок не найден!")
            return
        
        markup = types.InlineKeyboardMarkup()
        markup.add(
            types.InlineKeyboardButton("✅ Купить", callback_data=f"buy_{message.from_user.id}_{grams}_{price}"),
            types.InlineKeyboardButton("❌ Нет", callback_data="cancel")
        )
        
        bot.send_message(message.chat.id, f"✅ Предложение @{target} отправлено!")
        bot.send_message(buyer[0], f"💱 @{user[1]} продаёт *{grams}гр* за *{price:,}₽*", parse_mode="Markdown", reply_markup=markup)
    except:
        bot.send_message(message.chat.id, "❌ Формат: 2.5 5000 @user")

def process_ban(message):
    try:
        username = message.text.replace('@', '').strip()
        cursor.execute("UPDATE users SET is_banned = 1 WHERE username = ?", (username,))
        conn.commit()
        bot.send_message(message.chat.id, f"✅ @{username} забанен")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка")

def process_unban(message):
    try:
        username = message.text.replace('@', '').strip()
        cursor.execute("UPDATE users SET is_banned = 0 WHERE username = ?", (username,))
        conn.commit()
        bot.send_message(message.chat.id, f"✅ @{username} разбанен")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка")

def process_give(message):
    try:
        parts = message.text.split()
        username = parts[0].replace('@', '').strip()
        grams = float(parts[1])
        cursor.execute("UPDATE users SET grams = grams + ? WHERE username = ?", (grams, username))
        conn.commit()
        bot.send_message(message.chat.id, f"✅ @{username} +{grams}гр")
    except:
        bot.send_message(message.chat.id, "❌ Формат: @user 10")

def process_reset(message):
    try:
        username = message.text.replace('@', '').strip()
        cursor.execute("UPDATE users SET grams=0, money=0, level=1, is_growing=0, last_smoke_time=0, grow_start_time=0, grow_end_time=0 WHERE username=?", (username,))
        conn.commit()
        bot.send_message(message.chat.id, f"✅ @{username} обнулён")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка")

@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    if call.data.startswith("buy_"):
        _, seller_id, grams, price = call.data.split("_")
        seller_id = int(seller_id)
        grams = float(grams)
        price = int(price)
        buyer_id = call.from_user.id
        
        buyer = get_user(buyer_id)
        seller = get_user(seller_id)
        
        if buyer[3] < price:
            bot.answer_callback_query(call.id, "❌ Мало денег!", show_alert=True)
            return
        
        if seller[2] < grams:
            bot.answer_callback_query(call.id, "❌ Товара нет!", show_alert=True)
            return
        
        cursor.execute("UPDATE users SET grams = grams - ?, money = money + ? WHERE user_id = ?", (grams, price, seller_id))
        cursor.execute("UPDATE users SET grams = grams + ?, money = money - ? WHERE user_id = ?", (grams, price, buyer_id))
        conn.commit()
        
        bot.edit_message_text(f"✅ Сделка: {grams}гр за {price:,}₽", call.message.chat.id, call.message.message_id)
        bot.send_message(seller_id, f"💰 @{buyer[1]} купил у вас {grams}гр за {price:,}₽")
    
    eli
