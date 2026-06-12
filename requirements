# PixelBot

import os
import json
import random
import time
import logging
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Файл для хранения данных пользователей и цен
DATA_FILE = "game_data.json"
ADMIN_IDS = {8237260298}  # Замените на ваш Telegram ID (можно узнать у @userinfobot)

# Структура данных по умолчанию
DEFAULT_PRICES = {
    "house": 10000,
    "car": 5000,
    "computer": 2000,
    "phone": 800,
    "fun_cinema": 300,
    "fun_restaurant": 500,
    "fun_game": 150
}

DEFAULT_USER_DATA = {
    "balance": 1000,          # начальный капитал
    "happiness": 50,          # счастье (0-100)
    "inventory": {},          # { "house": 1, "car": 0, ... }
    "last_work": None,        # timestamp последней работы
    "last_daily": None,
    "work_streak": 0,
    "total_earned": 0
}

# Загрузка/сохранение данных
def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
            # Обратная совместимость: убедимся, что есть все ключи
            if "prices" not in data:
                data["prices"] = DEFAULT_PRICES.copy()
            if "users" not in data:
                data["users"] = {}
            return data
    else:
        return {"users": {}, "prices": DEFAULT_PRICES.copy()}

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# Глобальный объект данных (будет инициализирован в main)
game_data = None

# --- Вспомогательные функции ---
def get_user(user_id):
    uid = str(user_id)
    if uid not in game_data["users"]:
        game_data["users"][uid] = DEFAULT_USER_DATA.copy()
        game_data["users"][uid]["inventory"] = {}  # отдельно, чтобы не было ссылки
        save_data(game_data)
    return game_data["users"][uid]

def save_user(user_id):
    save_data(game_data)

def is_admin(user_id):
    return user_id in ADMIN_IDS

# --- Команды бота ---

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    get_user(user_id)  # инициализация
    keyboard = [
        [InlineKeyboardButton("💰 Финансы", callback_data="finance")],
        [InlineKeyboardButton("💼 Работа", callback_data="work")],
        [InlineKeyboardButton("🏠 Имущество", callback_data="property")],
        [InlineKeyboardButton("🎉 Развлечения", callback_data="fun")],
        [InlineKeyboardButton("🏪 Магазин / Цены", callback_data="shop")],
        [InlineKeyboardButton("📊 Статистика", callback_data="stats")]
    ]
    await update.message.reply_text(
        "🏙️ Добро пожаловать в симулятор жизни!\n\n"
        "У тебя есть деньги, которые можно зарабатывать, тратить на имущество и развлечения.\n"
        "💰 Баланс влияет на твою жизнь, а счастье – на доход от работы.\n\n"
        "Используй кнопки меню:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    data = query.data

    if data == "finance":
        await show_balance(update, context, user_id, query)
    elif data == "work":
        await work(update, context, user_id, query)
    elif data == "property":
        await show_property(update, context, user_id, query)
    elif data == "fun":
        await show_fun_menu(update, context, user_id, query)
    elif data == "shop":
        await show_shop(update, context, user_id, query)
    elif data == "stats":
        await show_stats(update, context, user_id, query)
    elif data.startswith("buy_"):
        item = data[4:]
        await buy_item(update, context, user_id, query, item)
    elif data.startswith("fun_"):
        fun_type = data[4:]
        await do_fun(update, context, user_id, query, fun_type)
    elif data == "daily":
        await daily_bonus(update, context, user_id, query)
    elif data == "back_to_menu":
        await show_main_menu(query)

async def show_balance(update, update_context, user_id, query):
    user = get_user(user_id)
    text = (f"💰 Ваш баланс: {user['balance']} монет\n"
            f"😊 Уровень счастья: {user['happiness']} / 100\n"
            f"💼 Заработано всего: {user['total_earned']} монет")
    keyboard = [[InlineKeyboardButton("◀️ Главное меню", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def work(update, update_context, user_id, query):
    user = get_user(user_id)
    now = time.time()
    cooldown = 60 * 5  # 5 минут между работой

    if user["last_work"] and (now - user["last_work"]) < cooldown:
        remaining = int(cooldown - (now - user["last_work"]))
        await query.edit_message_text(f"⏳ Вы устали! Отдохните {remaining // 60} мин {remaining % 60} сек.\n"
                                      f"Можете развлечься, чтобы повысить счастье.",
                                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]]))
        return

    # Доход зависит от счастья (база 200 + бонус до 200 от счастья)
    base_salary = 200
    happiness_bonus = int(user["happiness"] * 2)  # 0..200
    salary = base_salary + happiness_bonus + random.randint(-20, 20)
    user["balance"] += salary
    user["total_earned"] += salary
    user["last_work"] = now
    user["work_streak"] = user.get("work_streak", 0) + 1
    save_user(user_id)

    text = (f"💼 Вы отработали смену и заработали {salary} монет.\n"
            f"💰 Баланс: {user['balance']} монет\n"
            f"😊 Счастье влияет на зарплату: +{happiness_bonus} бонус.\n"
            f"📈 Рабочая серия: {user['work_streak']} дней подряд")
    keyboard = [[InlineKeyboardButton("◀️ Меню", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def show_property(update, update_context, user_id, query):
    user = get_user(user_id)
    inv = user["inventory"]
    prices = game_data["prices"]
    text = "🏠 Ваше имущество:\n"
    for item, price in prices.items():
        if item.startswith("fun_"):  # пропускаем развлечения
            continue
        count = inv.get(item, 0)
        if count > 0:
            text += f"• {item.capitalize()}: {count} шт. (цена покупки/продажи: {price} / {int(price*0.7)})\n"
    if not any(count > 0 for k, count in inv.items() if not k.startswith("fun_")):
        text += "У вас пока нет имущества. Купите в магазине!\n"
    text += "\n🏪 Для покупки/продажи используйте раздел «Магазин / Цены»."
    keyboard = [[InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def show_fun_menu(update, update_context, user_id, query):
    prices = game_data["prices"]
    keyboard = []
    for fun_item in ["cinema", "restaurant", "game"]:
        key = f"fun_{fun_item}"
        price = prices.get(key, 100)
        keyboard.append([InlineKeyboardButton(f"🎬 {fun_item.capitalize()} ({price}💰)", callback_data=f"fun_{fun_item}")])
    keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")])
    await query.edit_message_text("🎉 Выберите развлечение:\n"
                                  "Оно повысит ваше счастье, но потребует денег.",
                                  reply_markup=InlineKeyboardMarkup(keyboard))

async def do_fun(update, update_context, user_id, query, fun_type):
    user = get_user(user_id)
    prices = game_data["prices"]
    price_key = f"fun_{fun_type}"
    cost = prices.get(price_key, 100)

    if user["balance"] < cost:
        await query.edit_message_text(f"❌ Не хватает денег! Нужно {cost} монет, у вас {user['balance']}.",
                                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="fun")]]))
        return

    # Эффект на счастье и энергию
    happiness_gain = 0
    if fun_type == "cinema":
        happiness_gain = random.randint(5, 15)
        message = "🎬 Вы сходили в кино, отдохнули душой."
    elif fun_type == "restaurant":
        happiness_gain = random.randint(10, 20)
        message = "🍽️ Вкусный ужин в ресторане поднял настроение."
    else:  # game
        happiness_gain = random.randint(8, 18)
        message = "🎮 Игровая сессия подарила заряд бодрости."

    user["balance"] -= cost
    user["happiness"] = min(100, user["happiness"] + happiness_gain)
    save_user(user_id)

    text = (f"{message} +{happiness_gain} счастья.\n"
            f"💰 Остаток: {user['balance']} монет\n"
            f"😊 Текущее счастье: {user['happiness']}")
    keyboard = [[InlineKeyboardButton("◀️ К развлечениям", callback_data="fun")],
                [InlineKeyboardButton("Главное меню", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def show_shop(update, update_context, user_id, query):
    prices = game_data["prices"]
    keyboard = []
    # Показываем только товары (не развлечения)
    for item, price in prices.items():
        if not item.startswith("fun_"):
            keyboard.append([InlineKeyboardButton(f"🏷️ {item.capitalize()} - {price}💰", callback_data=f"buy_{item}")])
    keyboard.append([InlineKeyboardButton("📈 Изменить цены (админ)", callback_data="admin_prices")])
    keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")])
    await query.edit_message_text("🏪 Магазин:\nКупить предмет (влияет на престиж и потом можно продать):",
                                  reply_markup=InlineKeyboardMarkup(keyboard))

async def buy_item(update, update_context, user_id, query, item):
    user = get_user(user_id)
    prices = game_data["prices"]
    if item not in prices:
        await query.edit_message_text("❌ Товар не найден.", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="shop")]]))
        return
    price = prices[item]
    if user["balance"] < price:
        await query.edit_message_text(f"❌ Недостаточно средств! Нужно {price}, у вас {user['balance']}.",
                                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="shop")]]))
        return

    user["balance"] -= price
    inv = user["inventory"]
    inv[item] = inv.get(item, 0) + 1
    save_user(user_id)

    text = (f"✅ Вы купили {item.capitalize()} за {price} монет.\n"
            f"💰 Остаток: {user['balance']}\n"
            f"Имущество можно продать в будущем за 70% цены.")
    keyboard = [[InlineKeyboardButton("◀️ В магазин", callback_data="shop")],
                [InlineKeyboardButton("Главное меню", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def show_stats(update, update_context, user_id, query):
    user = get_user(user_id)
    inv = user["inventory"]
    total_items = sum(inv.values())
    # Подсчет общей стоимости имущества (по текущим ценам)
    total_value = sum(game_data["prices"].get(item, 0) * count for item, count in inv.items() if not item.startswith("fun_"))

    text = (f"📊 Статистика игрока:\n"
            f"💰 Денег: {user['balance']}\n"
            f"😊 Счастье: {user['happiness']} / 100\n"
            f"🏠 Кол-во имущества: {total_items} (оцен. стоимость {total_value} монет)\n"
            f"💼 Всего заработано: {user['total_earned']}\n"
            f"📈 Рабочая серия: {user['work_streak']} раз")
    keyboard = [[InlineKeyboardButton("🎁 Ежедневный бонус", callback_data="daily")],
                [InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]]
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

async def daily_bonus(update, update_context, user_id, query):
    user = get_user(user_id)
    now = datetime.now()
    last = user["last_daily"]
    if last:
        last_date = datetime.fromisoformat(last)
        if (now - last_date).days < 1:
            await query.edit_message_text("🎁 Вы уже получали ежедневный бонус сегодня. Возвращайтесь завтра!",
                                          reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]]))
            return
    bonus = random.randint(300, 600)
    user["balance"] += bonus
    user["last_daily"] = now.isoformat()
    save_user(user_id)
    await query.edit_message_text(f"🎉 Ежедневный бонус: +{bonus} монет! Баланс: {user['balance']}",
                                  reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Меню", callback_data="back_to_menu")]]))

# --- Админ-функции для изменения ценовой политики ---
async def admin_prices(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if not is_admin(user_id):
        await update.message.reply_text("⛔ Только администратор может менять цены.")
        return
    if not context.args:
        await update.message.reply_text("Использование: /set_price <товар> <цена>\n"
                                        "Доступные товары: house, car, computer, phone, fun_cinema, fun_restaurant, fun_game\n"
                                        "Пример: /set_price house 15000")
        return
    if len(context.args) != 2:
        await update.message.reply_text("Нужно указать название товара и цену.")
        return
    item, price_str = context.args
    try:
        price = int(price_str)
        if price < 0:
            raise ValueError
    except ValueError:
        await update.message.reply_text("Цена должна быть неотрицательным целым числом.")
        return
    if item not in game_data["prices"]:
        await update.message.reply_text(f"Товар '{item}' не найден. Список: {', '.join(game_data['prices'].keys())}")
        return
    game_data["prices"][item] = price
    save_data(game_data)
    await update.message.reply_text(f"✅ Цена на {item} установлена в {price} монет.\n"
                                    f"Теперь цены в магазине обновлены.")

async def show_prices(update: Update, context: ContextTypes.DEFAULT_TYPE):
    prices = game_data["prices"]
    text = "📋 Текущие цены (ценовая политика):\n"
    for item, price in prices.items():
        text += f"• {item.replace('fun_', '')}: {price} монет\n"
    await update.message.reply_text(text)

async def show_main_menu(query):
    keyboard = [
        [InlineKeyboardButton("💰 Финансы", callback_data="finance")],
        [InlineKeyboardButton("💼 Работа", callback_data="work")],
        [InlineKeyboardButton("🏠 Имущество", callback_data="property")],
        [InlineKeyboardButton("🎉 Развлечения", callback_data="fun")],
        [InlineKeyboardButton("🏪 Магазин / Цены", callback_data="shop")],
        [InlineKeyboardButton("📊 Статистика", callback_data="stats")]
    ]
    await query.edit_message_text("🏙️ Главное меню:", reply_markup=InlineKeyboardMarkup(keyboard))

async def unknown_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Неизвестная команда. Используйте /start")

def main():
    global game_data
    game_data = load_data()

    TOKEN = os.getenv("8675720922:AAFauNR5YY_VThTFaJ_m1Qwz-DZyg6MQsos")
    if not TOKEN:
        raise ValueError("Установите переменную окружения TELEGRAM_BOT_TOKEN")

    app = Application.builder().token(TOKEN).build()

    # Команды
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("set_price", admin_prices))
    app.add_handler(CommandHandler("prices", show_prices))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, unknown_command))

    logger.info("Бот запущен...")
    app.run_polling()

if __name__ == "__main__":
    main()
