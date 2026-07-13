Вот код без комментариев, скопируй и замени полностью свой bot.py.

```python
import sqlite3
import logging
from datetime import date, timedelta
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

TOKEN = "8688072352:AAHTUTJNaxsiV_d09vb1JnyqR6LiWjo7di4"

logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

DB_NAME = "calorie_bot.db"

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            daily_norm INTEGER DEFAULT 2000,
            weight REAL DEFAULT 70.0
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS food_base (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE,
            calories_per_100g REAL
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS activities_base (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE,
            calories_per_kg_per_hour REAL
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS daily_log (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            log_date TEXT,
            type TEXT,
            description TEXT,
            amount REAL,
            calories REAL
        )
    """)
    if cur.execute("SELECT COUNT(*) FROM food_base").fetchone()[0] == 0:
        default_food = [
            ("гречка", 110),
            ("рис", 116),
            ("курица вареная", 170),
            ("яблоко", 52),
            ("банан", 95),
            ("хлеб белый", 265),
            ("молоко 2.5%", 52),
            ("яйцо вареное", 155),
            ("овсянка на воде", 88),
            ("творог 5%", 145),
        ]
        cur.executemany("INSERT OR IGNORE INTO food_base (name, calories_per_100g) VALUES (?, ?)", default_food)

    if cur.execute("SELECT COUNT(*) FROM activities_base").fetchone()[0] == 0:
        default_act = [
            ("бег", 8.0),
            ("ходьба", 3.2),
            ("плавание", 7.0),
            ("велосипед", 6.0),
            ("силовая тренировка", 5.0),
            ("йога", 2.5),
        ]
        cur.executemany("INSERT OR IGNORE INTO activities_base (name, calories_per_kg_per_hour) VALUES (?, ?)", default_act)

    conn.commit()
    conn.close()

def get_user(user_id):
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT daily_norm, weight FROM users WHERE user_id = ?", (user_id,))
    row = cur.fetchone()
    if not row:
        cur.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        conn.commit()
        row = (2000, 70.0)
    conn.close()
    return row

def set_norm(user_id, norm):
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("INSERT OR REPLACE INTO users (user_id, daily_norm, weight) VALUES (?, ?, COALESCE((SELECT weight FROM users WHERE user_id = ?), 70.0))",
                (user_id, norm, user_id))
    conn.commit()
    conn.close()

def set_weight(user_id, weight):
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("INSERT OR REPLACE INTO users (user_id, daily_norm, weight) VALUES (?, COALESCE((SELECT daily_norm FROM users WHERE user_id = ?), 2000), ?)",
                (user_id, user_id, weight))
    conn.commit()
    conn.close()

main_keyboard = ReplyKeyboardMarkup(
    [
        ["/today", "/progress"],
        ["/search", "/add_product"],
        ["/once", "/activities"],
        ["/set_norm", "/set_weight"],
        ["/undo", "/clear", "/week"],
        ["/help", "/cancel", "/start"]
    ],
    resize_keyboard=True
)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    await update.message.reply_text(
        f"👋 Привет, {user.first_name}!\n"
        "Я помогу следить за калориями.\n"
        "Нажми /help, чтобы увидеть все команды, или просто начни вводить продукты.",
        reply_markup=main_keyboard
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = (
        "📋 <b>Доступные команды:</b>\n\n"
        "🍽 <b>Еда</b>\n"
        "• <code>гречка 150</code> – добавить порцию продукта (название и граммы)\n"
        "• /search – узнать калорийность продукта (без добавления в дневник)\n"
        "• /add_product – добавить новый продукт в общий справочник\n"
        "• /once – добавить разовое блюдо с калориями (не сохраняется)\n\n"
        "🏃 <b>Активности</b>\n"
        "• /activities – показать справочник активностей с расходом калорий\n"
        "• /act – записать тренировку (например, /act бег 30)\n\n"
        "📅 <b>Дневник</b>\n"
        "• /today – список всех приёмов пищи и активностей за сегодня\n"
        "• /progress – шкала выполнения дневной нормы\n"
        "• /week – статистика за последние 7 дней\n\n"
        "⚙️ <b>Настройки</b>\n"
        "• /set_norm – установить дневную норму калорий\n"
        "• /set_weight – сохранить свой вес (для точного расчёта активностей)\n\n"
        "🗑 <b>Управление</b>\n"
        "• /undo – отменить последнюю запись\n"
        "• /undo 3 – отменить запись номер 3 из списка /today\n"
        "• /clear – полностью очистить сегодняшний день\n"
        "• /cancel – выйти из режима ожидания (если бот что-то просит)\n\n"
        "❓ /help – показать эту подсказку"
    )
    await update.message.reply_html(text, reply_markup=main_keyboard)

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if context.user_data.get('awaiting_input'):
        context.user_data.pop('awaiting_input', None)
        await update.message.reply_text("✅ Действие отменено. Можно продолжать.")
    else:
        await update.message.reply_text("Нет активного действия для отмены.")

async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    state = context.user_data.get('awaiting_input')
    if state:
        text = update.message.text.strip()
        if state == 'add_product':
            await process_add_product_input(update, context, text)
        elif state == 'set_norm':
            await process_set_norm_input(update, context, text)
        elif state == 'set_weight':
            await process_set_weight_input(update, context, text)
        elif state == 'search':
            await process_search_input(update, context, text)
        elif state == 'act':
            await process_act_input(update, context, text)
        elif state == 'once':
            await process_once_input(update, context, text)
        elif state == 'undo':
            await process_undo_input(update, context, text)
        else:
            await update.message.reply_text("Неизвестное состояние. Используйте /cancel.")
            context.user_data.pop('awaiting_input', None)
        return
    await handle_food_text(update, context)

async def handle_food_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    parts = text.split()
    if len(parts) < 2:
        return
    try:
        weight = float(parts[-1])
    except ValueError:
        return
    product_name = " ".join(parts[:-1]).lower()

    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT calories_per_100g FROM food_base WHERE LOWER(name) = ?", (product_name,))
    row = cur.fetchone()
    if row:
        kcal_per_100 = row[0]
        total_kcal = (kcal_per_100 / 100) * weight
        today = str(date.today())
        user_id = update.effective_user.id
        cur.execute("INSERT INTO daily_log (user_id, log_date, type, description, amount, calories) VALUES (?, ?, 'food', ?, ?, ?)",
                    (user_id, today, f"{product_name} ({weight}г)", weight, total_kcal))
        conn.commit()
        await update.message.reply_text(f"✅ Добавлено: {product_name} {weight} г = {total_kcal:.1f} ккал")
    else:
        await update.message.reply_text(
            f"❌ Продукт <b>{product_name}</b> не найден в справочнике.\n"
            "Используй /search для поиска или /add_product, чтобы добавить новый.",
            parse_mode="HTML"
        )
    conn.close()

async def process_add_product_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    parts = text.rsplit(maxsplit=1)
    if len(parts) != 2:
        await update.message.reply_text("❌ Нужно ввести: <название продукта> <калорийность на 100г>\nПример: гречка 110")
        return
    name, kcal_str = parts
    try:
        kcal = float(kcal_str)
    except ValueError:
        await update.message.reply_text("❌ Калорийность должна быть числом. Попробуйте снова.")
        return
    if kcal <= 0:
        await update.message.reply_text("❌ Калорийность должна быть положительной.")
        return
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    try:
        cur.execute("INSERT INTO food_base (name, calories_per_100g) VALUES (?, ?)", (name.lower(), kcal))
        conn.commit()
        await update.message.reply_text(f"✅ Продукт <b>{name}</b> добавлен: {kcal} ккал/100г.", parse_mode="HTML")
    except sqlite3.IntegrityError:
        await update.message.reply_text("❌ Такой продукт уже существует в справочнике.")
    conn.close()
    context.user_data.pop('awaiting_input', None)

async def process_set_norm_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    try:
        norm = int(text)
    except ValueError:
        await update.message.reply_text("❌ Норма должна быть целым числом. Попробуйте ещё раз:")
        return
    if norm <= 0:
        await update.message.reply_text("❌ Норма должна быть положительной. Попробуйте ещё раз:")
        return
    set_norm(update.effective_user.id, norm)
    await update.message.reply_text(f"✅ Дневная норма установлена: {norm} ккал")
    context.user_data.pop('awaiting_input', None)

async def process_set_weight_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    try:
        weight = float(text)
    except ValueError:
        await update.message.reply_text("❌ Вес должен быть числом. Попробуйте ещё раз:")
        return
    if weight <= 0:
        await update.message.reply_text("❌ Вес должен быть положительным. Попробуйте ещё раз:")
        return
    set_weight(update.effective_user.id, weight)
    await update.message.reply_text(f"✅ Ваш вес сохранён: {weight} кг")
    context.user_data.pop('awaiting_input', None)

async def process_search_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    name = text.lower()
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT name, calories_per_100g FROM food_base WHERE LOWER(name) = ?", (name,))
    row = cur.fetchone()
    if row:
        await update.message.reply_text(f"🔍 {row[0]}: {row[1]} ккал на 100 г")
    else:
        cur.execute("SELECT name, calories_per_100g FROM food_base WHERE name LIKE ?", (f"%{name}%",))
        rows = cur.fetchall()
        if rows:
            msg = "Найдены похожие продукты:\n" + "\n".join(f"• {r[0]}: {r[1]} ккал/100г" for r in rows[:5])
            await update.message.reply_text(msg)
        else:
            await update.message.reply_text("Ничего не найдено.")
    conn.close()
    context.user_data.pop('awaiting_input', None)

async def process_act_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    parts = text.rsplit(maxsplit=1)
    if len(parts) != 2:
        await update.message.reply_text("❌ Введите: <активность> <минуты>\nПример: бег 30")
        return
    act_name, minutes_str = parts
    try:
        minutes = float(minutes_str)
    except ValueError:
        await update.message.reply_text("❌ Минуты должны быть числом. Попробуйте снова:")
        return
    if minutes <= 0:
        await update.message.reply_text("Минуты должны быть положительными.")
        return
    act_name = act_name.lower()
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT calories_per_kg_per_hour FROM activities_base WHERE LOWER(name) = ?", (act_name,))
    row = cur.fetchone()
    if not row:
        conn.close()
        await update.message.reply_text(f"Активность «{act_name}» не найдена. Посмотри /activities.")
        return
    cal_per_kg_per_hour = row[0]
    user_id = update.effective_user.id
    _, weight = get_user(user_id)
    hours = minutes / 60.0
    burned = cal_per_kg_per_hour * weight * hours
    today = str(date.today())
    cur.execute("INSERT INTO daily_log (user_id, log_date, type, description, amount, calories) VALUES (?, ?, 'activity', ?, ?, ?)",
                (user_id, today, f"{act_name} ({minutes} мин)", minutes, -burned))
    conn.commit()
    conn.close()
    await update.message.reply_text(f"🏃 Активность записана: {act_name} {minutes} мин. Сожжено ≈ {burned:.0f} ккал.")
    context.user_data.pop('awaiting_input', None)

async def process_once_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    parts = text.rsplit(maxsplit=1)
    if len(parts) != 2:
        await update.message.reply_text("❌ Введите: <название блюда> <калорийность>\nПример: пирожное 350")
        return
    name, kcal_str = parts
    try:
        kcal = float(kcal_str)
    except ValueError:
        await update.message.reply_text("❌ Калорийность должна быть числом. Попробуйте снова:")
        return
    if kcal < 0:
        await update.message.reply_text("Калорийность должна быть положительной.")
        return
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("INSERT INTO daily_log (user_id, log_date, type, description, amount, calories) VALUES (?, ?, 'food', ?, 0, ?)",
                (user_id, today, f"{name} (разовое)", kcal))
    conn.commit()
    conn.close()
    await update.message.reply_text(f"🍽 Разовое блюдо «{name}» ({kcal} ккал) добавлено в дневник.")
    context.user_data.pop('awaiting_input', None)

async def process_undo_input(update: Update, context: ContextTypes.DEFAULT_TYPE, text: str):
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT id FROM daily_log WHERE user_id = ? AND log_date = ? ORDER BY id", (user_id, today))
    entries = [row[0] for row in cur.fetchall()]
    if not entries:
        conn.close()
        await update.message.reply_text("Сегодня нет записей для отмены.")
        context.user_data.pop('awaiting_input', None)
        return
    try:
        n = int(text)
    except ValueError:
        await update.message.reply_text("❌ Номер записи должен быть целым числом. Попробуйте снова:")
        return
    if n < 1 or n > len(entries):
        await update.message.reply_text(f"Некорректный номер. Доступны номера 1–{len(entries)}.")
        return
    entry_id = entries[n-1]
    cur.execute("SELECT description, calories FROM daily_log WHERE id = ?", (entry_id,))
    desc, cal = cur.fetchone()
    cur.execute("DELETE FROM daily_log WHERE id = ?", (entry_id,))
    conn.commit()
    conn.close()
    sign = "+" if cal > 0 else ""
    await update.message.reply_text(f"🗑 Запись удалена: {desc} ({sign}{cal:.1f} ккал)")
    context.user_data.pop('awaiting_input', None)

async def search_product(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        context.user_data['awaiting_input'] = 'search'
        await update.message.reply_text("🔍 Введите название продукта для поиска:")
        return
    name = " ".join(context.args).lower()
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT name, calories_per_100g FROM food_base WHERE LOWER(name) = ?", (name,))
    row = cur.fetchone()
    if row:
        await update.message.reply_text(f"🔍 {row[0]}: {row[1]} ккал на 100 г")
    else:
        cur.execute("SELECT name, calories_per_100g FROM food_base WHERE name LIKE ?", (f"%{name}%",))
        rows = cur.fetchall()
        if rows:
            msg = "Найдены похожие продукты:\n" + "\n".join(f"• {r[0]}: {r[1]} ккал/100г" for r in rows[:5])
            await update.message.reply_text(msg)
        else:
            await update.message.reply_text("Ничего не найдено.")
    conn.close()

async def add_product(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) >= 2:
        try:
            kcal = float(context.args[-1])
        except ValueError:
            await update.message.reply_text("Ошибка: калорийность должна быть числом.")
            return
        name = " ".join(context.args[:-1]).lower()
        conn = sqlite3.connect(DB_NAME)
        cur = conn.cursor()
        try:
            cur.execute("INSERT INTO food_base (name, calories_per_100g) VALUES (?, ?)", (name, kcal))
            conn.commit()
            await update.message.reply_text(f"Продукт <b>{name}</b> добавлен: {kcal} ккал/100г.", parse_mode="HTML")
        except sqlite3.IntegrityError:
            await update.message.reply_text("Такой продукт уже существует в справочнике.")
        conn.close()
    else:
        context.user_data['awaiting_input'] = 'add_product'
        await update.message.reply_text("➕ Введите название продукта и калорийность на 100 г через пробел.\nПример: гречка 110")

async def once_food(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 2:
        context.user_data['awaiting_input'] = 'once'
        await update.message.reply_text("🍽 Введите название блюда и его калорийность через пробел (например: пирожное 350):")
        return
    try:
        kcal = float(context.args[-1])
    except ValueError:
        await update.message.reply_text("Ошибка: калорийность должна быть числом.")
        return
    name = " ".join(context.args[:-1])
    if kcal < 0:
        await update.message.reply_text("Калорийность должна быть положительной.")
        return
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("INSERT INTO daily_log (user_id, log_date, type, description, amount, calories) VALUES (?, ?, 'food', ?, 0, ?)",
                (user_id, today, f"{name} (разовое)", kcal))
    conn.commit()
    conn.close()
    await update.message.reply_text(f"🍽 Разовое блюдо «{name}» ({kcal} ккал) добавлено в дневник.")

async def list_activities(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    _, weight = get_user(user_id)
    try:
        conn = sqlite3.connect(DB_NAME)
        cur = conn.cursor()
        cur.execute("SELECT name, calories_per_kg_per_hour FROM activities_base")
        rows = cur.fetchall()
        conn.close()
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при обращении к базе: {e}")
        return
    if not rows:
        await update.message.reply_text("Справочник активностей пуст.")
        return
    msg = f"<b>Физические активности</b> (расход ккал/час для веса {weight} кг):\n\n"
    for name, cal_per_kg in rows:
        cal = cal_per_kg * weight
        msg += f"• {name}: {cal:.0f} ккал/ч\n"
    msg += "\nИспользуй /act &lt;активность&gt; &lt;минуты&gt; для записи."
    await update.message.reply_html(msg)

async def add_activity(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 2:
        context.user_data['awaiting_input'] = 'act'
        await update.message.reply_text("🏃 Введите активность и минуты через пробел (например: бег 30):")
        return
    try:
        minutes = float(context.args[-1])
    except ValueError:
        await update.message.reply_text("Ошибка: минуты должны быть числом.")
        return
    if minutes <= 0:
        await update.message.reply_text("Минуты должны быть положительными.")
        return
    act_name = " ".join(context.args[:-1]).lower()
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT calories_per_kg_per_hour FROM activities_base WHERE LOWER(name) = ?", (act_name,))
    row = cur.fetchone()
    if not row:
        conn.close()
        await update.message.reply_text(f"Активность «{act_name}» не найдена. Посмотри /activities.")
        return
    cal_per_kg_per_hour = row[0]
    user_id = update.effective_user.id
    _, weight = get_user(user_id)
    hours = minutes / 60.0
    burned = cal_per_kg_per_hour * weight * hours
    today = str(date.today())
    cur.execute("INSERT INTO daily_log (user_id, log_date, type, description, amount, calories) VALUES (?, ?, 'activity', ?, ?, ?)",
                (user_id, today, f"{act_name} ({minutes} мин)", minutes, -burned))
    conn.commit()
    conn.close()
    await update.message.reply_text(f"🏃 Активность записана: {act_name} {minutes} мин. Сожжено ≈ {burned:.0f} ккал.")

async def show_today(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT id, type, description, calories FROM daily_log WHERE user_id = ? AND log_date = ? ORDER BY id", (user_id, today))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        await update.message.reply_text("Сегодня ещё нет записей.")
        return
    total_food = 0
    total_activity = 0
    msg = "📅 <b>Сегодня:</b>\n\n"
    for idx, (row_id, typ, desc, cal) in enumerate(rows, 1):
        if cal > 0:
            total_food += cal
            msg += f"{idx}. 🍽 {desc}: +{cal:.1f} ккал\n"
        else:
            total_activity += abs(cal)
            msg += f"{idx}. 🏃 {desc}: -{abs(cal):.1f} ккал\n"
    net = total_food - total_activity
    msg += f"\n🍴 Получено: {total_food:.1f} ккал"
    msg += f"\n🔥 Сожжено: {total_activity:.1f} ккал"
    msg += f"\n⚖️ Чистый остаток: {net:.1f} ккал"
    await update.message.reply_html(msg)

async def show_progress(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    norm, _ = get_user(user_id)
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT SUM(calories) FROM daily_log WHERE user_id = ? AND log_date = ? AND type = 'food'", (user_id, today))
    food_sum = cur.fetchone()[0] or 0
    cur.execute("SELECT SUM(calories) FROM daily_log WHERE user_id = ? AND log_date = ? AND type = 'activity'", (user_id, today))
    act_sum = abs(cur.fetchone()[0] or 0)
    conn.close()
    net = food_sum - act_sum
    pct = min(100, (food_sum / norm) * 100) if norm > 0 else 0
    bar_len = 20
    filled = int(bar_len * pct / 100)
    bar = "█" * filled + "░" * (bar_len - filled)
    msg = (
        f"📊 <b>Прогресс по норме калорий</b>\n"
        f"Норма: {norm} ккал\n"
        f"Потреблено: {food_sum:.0f} ккал\n"
        f"[{bar}] {pct:.1f}%\n"
        f"Сожжено активно: {act_sum:.0f} ккал\n"
        f"Чистый остаток: {net:.0f} ккал"
    )
    await update.message.reply_html(msg)

async def set_daily_norm(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        context.user_data['awaiting_input'] = 'set_norm'
        await update.message.reply_text("⚖️ Введите дневную норму калорий (целое число):")
        return
    try:
        norm = int(context.args[0])
    except ValueError:
        await update.message.reply_text("Норма должна быть целым числом.")
        return
    if norm <= 0:
        await update.message.reply_text("Норма должна быть положительной.")
        return
    set_norm(update.effective_user.id, norm)
    await update.message.reply_text(f"✅ Дневная норма установлена: {norm} ккал")

async def set_user_weight(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        context.user_data['awaiting_input'] = 'set_weight'
        await update.message.reply_text("⚖️ Введите ваш вес в килограммах (например, 70):")
        return
    try:
        weight = float(context.args[0])
    except ValueError:
        await update.message.reply_text("Вес должен быть числом.")
        return
    if weight <= 0:
        await update.message.reply_text("Вес должен быть положительным.")
        return
    set_weight(update.effective_user.id, weight)
    await update.message.reply_text(f"✅ Ваш вес сохранён: {weight} кг")

async def undo_entry(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("SELECT id FROM daily_log WHERE user_id = ? AND log_date = ? ORDER BY id", (user_id, today))
    entries = [row[0] for row in cur.fetchall()]
    if not entries:
        conn.close()
        await update.message.reply_text("Сегодня нет записей для отмены.")
        return

    if not context.args:
        conn.close()
        context.user_data['awaiting_input'] = 'undo'
        await update.message.reply_text(f"🗑 Введите номер записи для отмены (от 1 до {len(entries)}):")
        return
    try:
        n = int(context.args[0])
    except ValueError:
        conn.close()
        await update.message.reply_text("Номер записи должен быть целым числом.")
        return
    if n < 1 or n > len(entries):
        conn.close()
        await update.message.reply_text(f"Некорректный номер. Доступны номера 1–{len(entries)}.")
        return
    entry_id = entries[n-1]
    cur.execute("SELECT description, calories FROM daily_log WHERE id = ?", (entry_id,))
    desc, cal = cur.fetchone()
    cur.execute("DELETE FROM daily_log WHERE id = ?", (entry_id,))
    conn.commit()
    conn.close()
    sign = "+" if cal > 0 else ""
    await update.message.reply_text(f"🗑 Запись удалена: {desc} ({sign}{cal:.1f} ккал)")

async def clear_today(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    today = str(date.today())
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute("DELETE FROM daily_log WHERE user_id = ? AND log_date = ?", (user_id, today))
    deleted = cur.rowcount
    conn.commit()
    conn.close()
    await update.message.reply_text(f"🗑 День очищен. Удалено записей: {deleted}")

async def week_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    today = date.today()
    start_date = today - timedelta(days=6)
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    stats = []
    for i in range(7):
        d = start_date + timedelta(days=i)
        d_str = d.isoformat()
        cur.execute("SELECT SUM(calories) FROM daily_log WHERE user_id = ? AND log_date = ?", (user_id, d_str))
        net = cur.fetchone()[0] or 0
        cur.execute("SELECT SUM(calories) FROM daily_log WHERE user_id = ? AND log_date = ? AND type = 'food'", (user_id, d_str))
        food = cur.fetchone()[0] or 0
        cur.execute("SELECT SUM(calories) FROM daily_log WHERE user_id = ? AND log_date = ? AND type = 'activity'", (user_id, d_str))
        act = abs(cur.fetchone()[0] or 0)
        stats.append((d_str, food, act, net))
    conn.close()
    msg = "<b>📆 Статистика за 7 дней:</b>\n\n"
    msg += "Дата          | Еда     | Актив   | Итого\n"
    msg += "------------------------------------------\n"
    for d_str, food, act, net in stats:
        msg += f"{d_str} | {food:7.0f} | {act:7.0f} | {net:7.0f}\n"
    await update.message.reply_html(msg)

def main():
    init_db()
    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("cancel", cancel))
    app.add_handler(CommandHandler("search", search_product))
    app.add_handler(CommandHandler("add_product", add_product))
    app.add_handler(CommandHandler("once", once_food))
    app.add_handler(CommandHandler("activities", list_activities))
    app.add_handler(CommandHandler("act", add_activity))
    app.add_handler(CommandHandler("today", show_today))
    app.add_handler(CommandHandler("progress", show_progress))
    app.add_handler(CommandHandler("set_norm", set_daily_norm))
    app.add_handler(CommandHandler("set_weight", set_user_weight))
    app.add_handler(CommandHandler("undo", undo_entry))
    app.add_handler(CommandHandler("clear", clear_today))
    app.add_handler(CommandHandler("week", week_stats))

    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))

    print("Бот запущен...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
```
