import os
import asyncio
import random
import html
import traceback
from datetime import datetime, date, timedelta

import aiosqlite
import httpx
from dotenv import load_dotenv

from aiogram import Bot, Dispatcher, F
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode, ChatMemberStatus
from aiogram.exceptions import TelegramBadRequest, TelegramForbiddenError
from aiogram.filters import Command, CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    Message,
    CallbackQuery,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")
TOOKEN_API_KEY = os.getenv("TOOKEN_API_KEY")
TOOKEN_BASE_URL = os.getenv("TOOKEN_BASE_URL", "https://tooken.club/v1")
TOOKEN_MODEL = os.getenv("TOOKEN_MODEL", "gpt-5.6-sol")

ADMIN_ID = int(os.getenv("ADMIN_ID", "8108428984"))
DAILY_BONUS_AMOUNT = 1

_raw_channel2 = os.getenv("SECOND_CHANNEL_USERNAME", "@krunchworld").strip()
SECOND_CHANNEL_USERNAME = _raw_channel2 if _raw_channel2.startswith("@") else f"@{_raw_channel2}"
TASK_CHANNEL_REWARD = int(os.getenv("TASK_CHANNEL_REWARD", "3"))

GAME_OUTCOMES = [(0, 40), (1, 35), (2, 18), (3, 7)]
DB_NAME = "tooken_bot.db"

bot = Bot(BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher(storage=MemoryStorage())


class PanelStates(StatesGroup):
    grant = State()
    premium = State()
    confirm = State()
    userinfo = State()
    broadcast = State()
    ban = State()
    reset_limit = State()
    add_admin = State()
    remove_admin = State()


# ==================== БАЗА ДАННЫХ ====================

async def init_db():
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                requests_used INTEGER DEFAULT 0,
                reset_date TEXT,
                premium_until TEXT,
                referrals INTEGER DEFAULT 0,
                referred_by INTEGER,
                bonus_requests INTEGER DEFAULT 0,
                last_bonus_date TEXT,
                banned INTEGER DEFAULT 0,
                task_channel2_done INTEGER DEFAULT 0,
                last_game_date TEXT
            )
        """)
        cursor = await db.execute("PRAGMA table_info(users)")
        columns = [row[1] for row in await cursor.fetchall()]
        migrations = {
            "last_bonus_date": "ALTER TABLE users ADD COLUMN last_bonus_date TEXT",
            "banned": "ALTER TABLE users ADD COLUMN banned INTEGER DEFAULT 0",
            "task_channel2_done": "ALTER TABLE users ADD COLUMN task_channel2_done INTEGER DEFAULT 0",
            "last_game_date": "ALTER TABLE users ADD COLUMN last_game_date TEXT",
        }
        for column, ddl in migrations.items():
            if column not in columns:
                await db.execute(ddl)

        await db.execute("""
            CREATE TABLE IF NOT EXISTS admins (user_id INTEGER PRIMARY KEY, added_by INTEGER, added_at TEXT)
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS settings (key TEXT PRIMARY KEY, value TEXT)
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS logs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                username TEXT,
                question TEXT,
                answer TEXT,
                timestamp TEXT
            )
        """)
        await db.commit()


async def get_user(user_id: int, username: str = ""):
    today = date.today().isoformat()
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
        user = await cursor.fetchone()

        if not user:
            await db.execute(
                "INSERT INTO users (user_id, username, reset_date) VALUES (?, ?, ?)",
                (user_id, username or "", today)
            )
            await db.commit()
            cursor = await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
            user = await cursor.fetchone()
        elif user[3] != today:
            await db.execute(
                "UPDATE users SET requests_used = 0, reset_date = ? WHERE user_id = ?",
                (today, user_id)
            )
            await db.commit()
            cursor = await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
            user = await cursor.fetchone()

        return user


async def update_user(user_id: int, query: str, params: tuple = ()):
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute(query, params)
        await db.commit()


async def all_user_ids() -> list[int]:
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT user_id FROM users")
        rows = await cursor.fetchall()
    return [row[0] for row in rows]


async def is_admin_user(user_id: int) -> bool:
    if user_id == ADMIN_ID:
        return True
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,))
        return await cursor.fetchone() is not None


async def all_admin_ids() -> list[int]:
    ids = {ADMIN_ID}
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT user_id FROM admins")
        rows = await cursor.fetchall()
    ids.update(row[0] for row in rows)
    return list(ids)


def truncate(text: str, limit: int = 800) -> str:
    return text if len(text) <= limit else text[:limit] + "… (обрезано)"


def premium_active(user) -> bool:
    if not user[4]:
        return False
    try:
        return datetime.fromisoformat(user[4]) > datetime.now()
    except ValueError:
        return False


def daily_limit(user) -> int:
    base = 10 if premium_active(user) else 2
    return base + user[5] + user[7]


def subscription_name(user) -> str:
    return "💎 Premium" if premium_active(user) else "🆓 Free"


def bonus_available(user) -> bool:
    return user[8] != date.today().isoformat()


def is_banned(user) -> bool:
    return bool(user[9])


def task_channel2_done(user) -> bool:
    return bool(user[10])


def game_available(user) -> bool:
    return user[11] != date.today().isoformat()


def progress_bar(used: int, limit: int, length: int = 10) -> str:
    if limit <= 0:
        limit = 1
    filled = min(length, round(length * used / limit))
    return "▓" * filled + "░" * (length - filled)


# ==================== КЛАВИАТУРЫ ====================

def main_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="✨ Новый запрос", callback_data="ask"),
            InlineKeyboardButton(text="🎁 Бонус", callback_data="daily_bonus")
        ],
        [
            InlineKeyboardButton(text="🎯 Задания", callback_data="tasks"),
            InlineKeyboardButton(text="🎲 Мини-игра", callback_data="minigame")
        ],
        [
            InlineKeyboardButton(text="👤 Профиль", callback_data="profile"),
            InlineKeyboardButton(text="🛍 Магазин", callback_data="shop")
        ],
        [
            InlineKeyboardButton(text="👥 Рефералы", callback_data="referrals"),
            InlineKeyboardButton(text="🆘 Помощь", callback_data="help")
        ],
        [
            InlineKeyboardButton(text="🔴 Закрыть", callback_data="close_menu")
        ]
    ])


def back_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="⬅️ В меню", callback_data="back_to_menu"),
            InlineKeyboardButton(text="🔴 Закрыть", callback_data="close_menu")
        ]
    ])


def tasks_keyboard(user):
    done = task_channel2_done(user)
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="📢 Открыть канал", url=f"https://t.me/{SECOND_CHANNEL_USERNAME.lstrip('@')}")
        ],
        [
            InlineKeyboardButton(text="✅ Выполнено" if done else "🔍 Проверить подписку", callback_data="task_channel2")
        ],
        [
            InlineKeyboardButton(text="⬅️ В меню", callback_data="back_to_menu"),
            InlineKeyboardButton(text="🔴 Закрыть", callback_data="close_menu")
        ]
    ])


def game_keyboard(user):
    if not game_available(user):
        return back_keyboard()
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="🎰 Крутить барабан!", callback_data="minigame_spin")
        ],
        [
            InlineKeyboardButton(text="⬅️ В меню", callback_data="back_to_menu"),
            InlineKeyboardButton(text="🔴 Закрыть", callback_data="close_menu")
        ]
    ])


# ==================== ТЕКСТЫ ====================

def welcome_text():
    return (
        "🤖 <b>ChatGPT</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "Умный AI-ассистент с живым веб-поиском.\n"
        "Отвечаю быстро, точно и по делу 🚀\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "Выберите действие ниже 👇"
    )


def profile_text(user):
    limit = daily_limit(user)
    bar = progress_bar(user[2], limit)
    return (
        "✨ <b>ВАШ ПРОФИЛЬ</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"🆔 ID: <code>{user[0]}</code>\n"
        f"💳 Тариф: <b>{subscription_name(user)}</b>\n"
        f"📅 Premium до: {user[4] or '—'}\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"📊 Запросы сегодня: <code>{bar}</code> {user[2]}/{limit}\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"👥 Приглашено друзей: <b>{user[5]}</b>\n"
        f"🎁 Бонусных запросов: <b>{user[7]}</b>\n"
        f"{'🎁 Бонус: доступен' if bonus_available(user) else '🎁 Бонус: получен ✅'}"
    )


def shop_text():
    return (
        "🛍 <b>МАГАЗИН</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "💎 <b>Premium — 20 ₽ / 30 дней</b>\n"
        "  • 10 запросов в день\n"
        "  • Приоритетный доступ\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "<b>Как оплатить:</b>\n"
        "1️⃣ Зарегистрируйтесь на STARVELL\n"
        "2️⃣ Оплатите по ссылке:\n"
        "https://starvell.com/offers/832e12a1-04ef-4636-9498-1fafa5f74bb9\n"
        "3️⃣ Напишите админу свой Telegram username\n"
        "4️⃣ Premium активируют вручную ✅"
    )


def referrals_text(link, user):
    return (
        "👥 <b>РЕФЕРАЛЬНАЯ СИСТЕМА</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "За каждого друга — <b>+1 запрос в день навсегда</b> 🔥\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"👥 Приглашено: <b>{user[5]}</b>\n\n"
        "🔗 Ваша ссылка:\n"
        f"<code>{link}</code>"
    )


def help_text():
    return (
        "🆘 <b>ПОМОЩЬ</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "✨ «Новый запрос» — задать вопрос AI\n"
        "🎁 «Бонус» — забрать +1 запрос раз в день\n"
        "🎯 «Задания» — разовые награды за действия\n"
        "🎲 «Мини-игра» — шанс выиграть до +3 запросов в день\n"
        "🆓 Free — 2 запроса в день\n"
        "💎 Premium — 10 запросов в день\n"
        "👥 Реферал — +1 запрос в день навсегда\n"
        "💳 Оплата подтверждается администратором вручную"
    )


def tasks_text(user):
    status = "✅ <b>Выполнено</b>" if task_channel2_done(user) else f"🎁 Награда: <b>+{TASK_CHANNEL_REWARD} запроса</b>"
    return (
        "🎯 <b>ЗАДАНИЯ</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"1️⃣ Подпишитесь на {SECOND_CHANNEL_USERNAME}\n"
        f"{status}\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "Подпишитесь и нажмите проверку 👇"
    )


def game_text(user):
    if not game_available(user):
        return "🎲 <b>МИНИ-ИГРА</b>\n━━━━━━━━━━━━━━━━━━\nВы уже играли сегодня 🎰\nВозвращайтесь завтра за новой попыткой! 🌙"
    return (
        "🎲 <b>МИНИ-ИГРА</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "Один спин в день — шанс выиграть бонусные запросы!\n\n"
        "🎯 Шансы:\n"
        "40% — пусто 😔\n"
        "35% — +1 запрос 🎉\n"
        "18% — +2 запроса 🔥\n"
        "7% — 💎 ДЖЕКПОТ +3 запроса!"
    )


# ==================== TOOKEN API ====================

async def ask_tooken(prompt: str) -> str:
    url = f"{TOOKEN_BASE_URL.rstrip('/')}/responses"
    headers = {
        "Authorization": f"Bearer {TOOKEN_API_KEY}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": TOOKEN_MODEL,
        "input": prompt,
        "web_search": "live"
    }

    async def send(body):
        async with httpx.AsyncClient(timeout=120) as client:
            return await client.post(url, json=body, headers=headers)

    response = await send(payload)

    # У этого API нестандартное поле — "web_search": "live". Судя по всему,
    # именно оно вызывает 400 ("параметры не подходят выбранной модели").
    # При любой ошибке 400 пробуем повторить запрос уже без него.
    if response.status_code == 400 and "web_search" in payload:
        print(f"[API WARNING] Запрос с web_search отклонён: {response.text[:1500]}")
        fallback_payload = {k: v for k, v in payload.items() if k != "web_search"}
        response = await send(fallback_payload)
        if response.status_code == 200:
            print("[API INFO] Успех без web_search — поле web_search не поддерживается этой моделью/API.")

    if response.status_code != 200:
        error_detail = f"Status: {response.status_code}\nBody: {response.text[:1500]}"
        print(f"[API ERROR] {error_detail}")
        raise Exception(f"API вернул ошибку {response.status_code}: {response.text[:1500]}")

    data = response.json()

    if isinstance(data.get("output_text"), str):
        return data["output_text"]
    if isinstance(data.get("text"), str):
        return data["text"]

    output = data.get("output")
    if isinstance(output, str):
        return output
    if isinstance(output, list):
        parts = []
        for item in output:
            if isinstance(item, dict):
                content = item.get("content", [])
                if isinstance(content, str):
                    parts.append(content)
                elif isinstance(content, list):
                    for block in content:
                        if isinstance(block, dict) and block.get("text"):
                            parts.append(block["text"])
        if parts:
            return "\n".join(parts)

    return "Не удалось распознать ответ API Tooken."


# ==================== ХЕНДЛЕРЫ ПОЛЬЗОВАТЕЛЕЙ ====================

@dp.message(CommandStart())
async def start_handler(message: Message):
    args = message.text.split(maxsplit=1)
    referrer_id = None
    if len(args) > 1 and args[1].startswith("ref_"):
        try:
            referrer_id = int(args[1].replace("ref_", ""))
        except ValueError:
            pass

    user = await get_user(message.from_user.id, message.from_user.username or "")
    if referrer_id and referrer_id != message.from_user.id and not user[6]:
        await update_user(
            message.from_user.id,
            "UPDATE users SET referred_by = ? WHERE user_id = ?",
            (referrer_id, message.from_user.id)
        )
        await get_user(referrer_id)
        await update_user(
            referrer_id,
            "UPDATE users SET referrals = referrals + 1 WHERE user_id = ?",
            (referrer_id,)
        )

    await message.answer(welcome_text(), reply_markup=main_keyboard())


@dp.callback_query(lambda c: not (c.data or "").startswith("panel_"))
async def callback_handler(callback: CallbackQuery, state: FSMContext):
    data = callback.data
    user = await get_user(callback.from_user.id, callback.from_user.username or "")

    # ===== МЕНЮ =====
    if data == "back_to_menu":
        await callback.message.delete()
        await callback.message.answer(welcome_text(), reply_markup=main_keyboard())
        await callback.answer()
        return

    if data == "close_menu":
        await state.clear()
        try:
            await callback.message.delete()
        except:
            pass
        await callback.answer()
        return

    # ===== ПОЛЬЗОВАТЕЛЬСКИЕ КНОПКИ =====
    if data == "profile":
        await callback.message.delete()
        await callback.message.answer(profile_text(user), reply_markup=back_keyboard())
        await callback.answer()
        return

    if data == "shop":
        await callback.message.delete()
        await callback.message.answer(shop_text(), reply_markup=back_keyboard())
        await callback.answer()
        return

    if data == "referrals":
        bot_info = await bot.get_me()
        link = f"https://t.me/{bot_info.username}?start=ref_{callback.from_user.id}"
        await callback.message.delete()
        await callback.message.answer(referrals_text(link, user), reply_markup=back_keyboard())
        await callback.answer()
        return

    if data == "help":
        await callback.message.delete()
        await callback.message.answer(help_text(), reply_markup=back_keyboard())
        await callback.answer()
        return

    if data == "daily_bonus":
        if not bonus_available(user):
            await callback.answer("🎁 Бонус за сегодня уже получен. Приходите завтра!", show_alert=True)
            return
        today = date.today().isoformat()
        await update_user(
            callback.from_user.id,
            "UPDATE users SET bonus_requests = bonus_requests + ?, last_bonus_date = ? WHERE user_id = ?",
            (DAILY_BONUS_AMOUNT, today, callback.from_user.id)
        )
        await callback.answer(f"🎉 +{DAILY_BONUS_AMOUNT} запрос начислен!", show_alert=True)
        user = await get_user(callback.from_user.id)
        await callback.message.delete()
        await callback.message.answer(profile_text(user), reply_markup=back_keyboard())
        return

    if data == "ask":
        if is_banned(user):
            await callback.answer("🚫 Вы заблокированы.", show_alert=True)
            return
        if user[2] >= daily_limit(user):
            await callback.answer("Запросы на сегодня закончились.", show_alert=True)
            return
        await callback.message.answer("✍️ Напишите ваш вопрос следующим сообщением.")
        await callback.answer()
        return

    if data == "tasks":
        await callback.message.delete()
        await callback.message.answer(tasks_text(user), reply_markup=tasks_keyboard(user))
        await callback.answer()
        return

    if data == "task_channel2":
        if task_channel2_done(user):
            await callback.answer("✅ Это задание уже выполнено.", show_alert=True)
            return
        try:
            member = await bot.get_chat_member(SECOND_CHANNEL_USERNAME, callback.from_user.id)
            subscribed = member.status in {ChatMemberStatus.MEMBER, ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.CREATOR}
        except:
            subscribed = False
        if not subscribed:
            await callback.answer("❌ Вы ещё не подписались на канал.", show_alert=True)
            return
        await update_user(
            callback.from_user.id,
            "UPDATE users SET bonus_requests = bonus_requests + ?, task_channel2_done = 1 WHERE user_id = ?",
            (TASK_CHANNEL_REWARD, callback.from_user.id)
        )
        await callback.answer(f"🎉 Задание выполнено! +{TASK_CHANNEL_REWARD} запроса.", show_alert=True)
        user = await get_user(callback.from_user.id)
        await callback.message.delete()
        await callback.message.answer(tasks_text(user), reply_markup=tasks_keyboard(user))
        return

    if data == "minigame":
        await callback.message.delete()
        await callback.message.answer(game_text(user), reply_markup=game_keyboard(user))
        await callback.answer()
        return

    if data == "minigame_spin":
        if not game_available(user):
            await callback.answer("Вы уже играли сегодня!", show_alert=True)
            return
        amounts = [a for a, _ in GAME_OUTCOMES]
        weights = [w for _, w in GAME_OUTCOMES]
        win = random.choices(amounts, weights=weights)[0]
        today = date.today().isoformat()
        if win > 0:
            await update_user(
                callback.from_user.id,
                "UPDATE users SET bonus_requests = bonus_requests + ?, last_game_date = ? WHERE user_id = ?",
                (win, today, callback.from_user.id)
            )
        else:
            await update_user(
                callback.from_user.id,
                "UPDATE users SET last_game_date = ? WHERE user_id = ?",
                (today, callback.from_user.id)
            )
        msgs = {0: "😔 Не повезло!", 1: "🎉 +1 запрос!", 2: "🔥 +2 запроса!", 3: "💎 ДЖЕКПОТ +3!"}
        await callback.answer(msgs[win], show_alert=True)
        user = await get_user(callback.from_user.id)
        await callback.message.delete()
        await callback.message.answer(game_text(user), reply_markup=game_keyboard(user))
        return


# ==================== АДМИН-ПАНЕЛЬ ====================

def panel_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="🔴 Закрыть", callback_data="panel_close")
        ],
        [
            InlineKeyboardButton(text="⚡ Выдать запросы", callback_data="panel_grant"),
            InlineKeyboardButton(text="💎 Выдать Premium", callback_data="panel_premium")
        ],
        [
            InlineKeyboardButton(text="✅ Подтвердить оплату", callback_data="panel_confirm"),
            InlineKeyboardButton(text="🔎 Инфо о юзере", callback_data="panel_userinfo")
        ],
        [
            InlineKeyboardButton(text="📊 Статистика", callback_data="panel_stats"),
            InlineKeyboardButton(text="📢 Рассылка", callback_data="panel_broadcast")
        ],
        [
            InlineKeyboardButton(text="🚫 Бан/Разбан", callback_data="panel_ban"),
            InlineKeyboardButton(text="🔄 Сброс лимита", callback_data="panel_reset_limit")
        ],
        [
            InlineKeyboardButton(text="➕ Добавить админа", callback_data="panel_add_admin"),
            InlineKeyboardButton(text="➖ Удалить админа", callback_data="panel_remove_admin")
        ],
        [
            InlineKeyboardButton(text="👀 Прослушка", callback_data="panel_listen")
        ],
        [
            InlineKeyboardButton(text="🏆 Топ рефералов", callback_data="panel_top")
        ]
    ])


def cancel_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="❌ Отмена", callback_data="panel_cancel")
        ]
    ])


@dp.message(Command("panel"))
async def panel_open(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    await state.clear()
    await message.answer(
        "🛠 <b>ПАНЕЛЬ АДМИНИСТРАТОРА</b>\n━━━━━━━━━━━━━━━━━━\nВыберите действие:",
        reply_markup=panel_keyboard()
    )


@dp.callback_query(lambda c: (c.data or "").startswith("panel_"))
async def panel_callbacks(callback: CallbackQuery, state: FSMContext):
    if not await is_admin_user(callback.from_user.id):
        await callback.answer()
        return

    data = callback.data

    # ===== ЗАКРЫТИЕ =====
    if data == "panel_close":
        await state.clear()
        await callback.message.delete()
        await callback.answer()
        return

    if data == "panel_cancel":
        await state.clear()
        await callback.message.delete()
        await callback.message.answer(
            "🛠 <b>ПАНЕЛЬ АДМИНИСТРАТОРА</b>\n━━━━━━━━━━━━━━━━━━\nВыберите действие:",
            reply_markup=panel_keyboard()
        )
        await callback.answer("Отменено")
        return

    # ===== ВЫДАТЬ ЗАПРОСЫ =====
    if data == "panel_grant":
        await state.set_state(PanelStates.grant)
        await callback.message.delete()
        await callback.message.answer(
            "⚡ <b>Выдать запросы</b>\n\nОтправьте: <code>ID количество</code>\nНапример: <code>123456789 5</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== ВЫДАТЬ PREMIUM =====
    if data == "panel_premium":
        await state.set_state(PanelStates.premium)
        await callback.message.delete()
        await callback.message.answer(
            "💎 <b>Выдать Premium</b>\n\nОтправьте: <code>ID дни</code>\nНапример: <code>123456789 30</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== ПОДТВЕРДИТЬ ОПЛАТУ =====
    if data == "panel_confirm":
        await state.set_state(PanelStates.confirm)
        await callback.message.delete()
        await callback.message.answer(
            "✅ <b>Подтвердить оплату (30 дней Premium)</b>\n\nОтправьте: <code>ID</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== ИНФО О ЮЗЕРЕ =====
    if data == "panel_userinfo":
        await state.set_state(PanelStates.userinfo)
        await callback.message.delete()
        await callback.message.answer(
            "🔎 <b>Инфо о пользователе</b>\n\nОтправьте: <code>ID</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== СТАТИСТИКА =====
    if data == "panel_stats":
        async with aiosqlite.connect(DB_NAME) as db:
            cursor = await db.execute("SELECT COUNT(*), COALESCE(SUM(requests_used), 0) FROM users")
            users_count, requests_count = await cursor.fetchone()
            cursor = await db.execute("SELECT COUNT(*) FROM users WHERE premium_until > ?", (datetime.now().isoformat(),))
            premium_count = (await cursor.fetchone())[0]
        await callback.message.delete()
        await callback.message.answer(
            f"📊 <b>СТАТИСТИКА БОТА</b>\n━━━━━━━━━━━━━━━━━━\n👥 Пользователей: <b>{users_count}</b>\n💎 Premium: <b>{premium_count}</b>\n📨 Запросов сегодня: <b>{requests_count}</b>",
            reply_markup=panel_keyboard()
        )
        await callback.answer()
        return

    # ===== РАССЫЛКА =====
    if data == "panel_broadcast":
        await state.set_state(PanelStates.broadcast)
        await callback.message.delete()
        await callback.message.answer(
            "📢 <b>Рассылка</b>\n\nОтправьте текст сообщения для всех пользователей.",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== БАН/РАЗБАН =====
    if data == "panel_ban":
        await state.set_state(PanelStates.ban)
        await callback.message.delete()
        await callback.message.answer(
            "🚫 <b>Бан / Разбан</b>\n\nОтправьте: <code>ID</code>\n(повторная отправка того же ID снимает бан)",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== СБРОС ЛИМИТА =====
    if data == "panel_reset_limit":
        await state.set_state(PanelStates.reset_limit)
        await callback.message.delete()
        await callback.message.answer(
            "🔄 <b>Сброс дневного лимита</b>\n\nОтправьте: <code>ID</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== ДОБАВИТЬ АДМИНА =====
    if data == "panel_add_admin":
        await state.set_state(PanelStates.add_admin)
        await callback.message.delete()
        await callback.message.answer(
            "➕ <b>Добавить админа</b>\n\nОтправьте: <code>ID</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== УДАЛИТЬ АДМИНА =====
    if data == "panel_remove_admin":
        await state.set_state(PanelStates.remove_admin)
        await callback.message.delete()
        await callback.message.answer(
            "➖ <b>Удалить админа</b>\n\nОтправьте: <code>ID</code>",
            reply_markup=cancel_keyboard()
        )
        await callback.answer()
        return

    # ===== ПРОСЛУШКА =====
    if data == "panel_listen":
        async with aiosqlite.connect(DB_NAME) as db:
            cursor = await db.execute(
                "SELECT user_id, username, question, answer, timestamp FROM logs ORDER BY id DESC LIMIT 20"
            )
            rows = await cursor.fetchall()
        await callback.message.delete()
        if not rows:
            await callback.message.answer(
                "👀 <b>ПРОСЛУШКА</b>\n━━━━━━━━━━━━━━━━━━\nПока нет записей.",
                reply_markup=panel_keyboard()
            )
            await callback.answer()
            return
        lines = ["👀 <b>ПРОСЛУШКА (последние 20 запросов)</b>", "━━━━━━━━━━━━━━━━━━"]
        for user_id, username, question, answer, timestamp in rows:
            name = f"@{username}" if username else f"ID {user_id}"
            lines.append(f"👤 {name} | {timestamp}")
            lines.append(f"❓ {truncate(question, 100)}")
            lines.append(f"💬 {truncate(answer, 100)}")
            lines.append("──────────────")
        await callback.message.answer("\n".join(lines), reply_markup=panel_keyboard())
        await callback.answer()
        return

    # ===== ТОП РЕФЕРАЛОВ =====
    if data == "panel_top":
        async with aiosqlite.connect(DB_NAME) as db:
            cursor = await db.execute(
                "SELECT user_id, username, referrals FROM users ORDER BY referrals DESC LIMIT 5"
            )
            rows = await cursor.fetchall()
        await callback.message.delete()
        medals = ["🥇", "🥈", "🥉", "🏅", "🏅"]
        lines = ["🏆 <b>ТОП РЕФЕРАЛОВ</b>", "━━━━━━━━━━━━━━━━━━"]
        if not rows:
            lines.append("Пока нет данных.")
        else:
            for i, (user_id, username, referrals) in enumerate(rows):
                name = f"@{username}" if username else f"ID {user_id}"
                lines.append(f"{medals[i]} {name} — {referrals} реф.")
        await callback.message.answer("\n".join(lines), reply_markup=panel_keyboard())
        await callback.answer()
        return


# ==================== ОБРАБОТЧИКИ FSM ====================

@dp.message(PanelStates.grant)
async def grant_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    parts = message.text.split()
    if len(parts) != 2 or not parts[0].lstrip("-").isdigit() or not parts[1].lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID количество</code>", reply_markup=cancel_keyboard())
        return
    user_id, amount = int(parts[0]), int(parts[1])
    await get_user(user_id)
    await update_user(user_id, "UPDATE users SET bonus_requests = bonus_requests + ? WHERE user_id = ?", (amount, user_id))
    await state.clear()
    await message.answer(f"✅ Пользователю <code>{user_id}</code> начислено <b>{amount}</b> запрос(ов).", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, f"🎁 Администратор начислил вам <b>+{amount}</b> запрос(ов)!")
    except:
        pass


@dp.message(PanelStates.premium)
async def premium_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    parts = message.text.split()
    if len(parts) != 2 or not parts[0].lstrip("-").isdigit() or not parts[1].lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID дни</code>", reply_markup=cancel_keyboard())
        return
    user_id, days = int(parts[0]), int(parts[1])
    until = (datetime.now() + timedelta(days=days)).isoformat(timespec="seconds")
    await update_user(user_id, "UPDATE users SET premium_until = ? WHERE user_id = ?", (until, user_id))
    await state.clear()
    await message.answer(f"✅ Premium выдан <code>{user_id}</code> на <b>{days}</b> дней.", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, f"💎 Вам активирован <b>Premium</b> на {days} дней!")
    except:
        pass


@dp.message(PanelStates.confirm)
async def confirm_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    until = (datetime.now() + timedelta(days=30)).isoformat(timespec="seconds")
    await update_user(user_id, "UPDATE users SET premium_until = ? WHERE user_id = ?", (until, user_id))
    await state.clear()
    await message.answer(f"✅ Оплата подтверждена. Premium активирован для <code>{user_id}</code> на 30 дней.", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, "💎 Оплата подтверждена! Premium активирован на 30 дней.")
    except:
        pass


@dp.message(PanelStates.userinfo)
async def userinfo_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    user = await get_user(user_id)
    await state.clear()
    await message.answer(
        f"🔎 <b>ИНФО О ПОЛЬЗОВАТЕЛЕ</b>\n━━━━━━━━━━━━━━━━━━\n"
        f"ID: <code>{user[0]}</code>\nUsername: @{user[1] or 'не указан'}\n"
        f"Тариф: {subscription_name(user)}\nPremium до: {user[4] or '—'}\n"
        f"Запросов сегодня: {user[2]}/{daily_limit(user)}\n"
        f"Бонусных запросов: {user[7]}\nРефералов: {user[5]}\n"
        f"Статус: {'🚫 Заблокирован' if is_banned(user) else '✅ Активен'}\n"
        f"Задание (2-й канал): {'✅ выполнено' if task_channel2_done(user) else '❌ не выполнено'}",
        reply_markup=panel_keyboard()
    )


@dp.message(PanelStates.broadcast)
async def broadcast_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    await state.clear()
    text = f"📣 <b>Администратор☑</b> : {message.text}"
    ids = await all_user_ids()
    sent, failed = 0, 0
    for user_id in ids:
        try:
            await bot.send_message(user_id, text)
            sent += 1
        except:
            failed += 1
        await asyncio.sleep(0.05)
    await message.answer(f"📢 Рассылка завершена.\n✅ Доставлено: {sent}\n⚠️ Ошибок: {failed}", reply_markup=panel_keyboard())


@dp.message(PanelStates.ban)
async def ban_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    user = await get_user(user_id)
    new_status = 0 if is_banned(user) else 1
    await update_user(user_id, "UPDATE users SET banned = ? WHERE user_id = ?", (new_status, user_id))
    await state.clear()
    await message.answer(
        f"{'🚫 Пользователь' if new_status else '✅ Пользователь'} <code>{user_id}</code> {'заблокирован.' if new_status else 'разблокирован.'}",
        reply_markup=panel_keyboard()
    )
    try:
        await bot.send_message(user_id, "🚫 Вы заблокированы администратором." if new_status else "✅ Вы разблокированы администратором.")
    except:
        pass


@dp.message(PanelStates.reset_limit)
async def reset_limit_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    await update_user(user_id, "UPDATE users SET requests_used = 0 WHERE user_id = ?", (user_id,))
    await state.clear()
    await message.answer(f"🔄 Лимит пользователя <code>{user_id}</code> сброшен.", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, "🔄 Администратор сбросил ваш дневной лимит запросов!")
    except:
        pass


@dp.message(PanelStates.add_admin)
async def add_admin_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    if user_id == ADMIN_ID:
        await message.answer("⚠️ Владелец бота уже администратор.", reply_markup=cancel_keyboard())
        return
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,))
        if await cursor.fetchone():
            await message.answer("⚠️ Уже администратор.", reply_markup=cancel_keyboard())
            return
        await db.execute("INSERT INTO admins (user_id, added_by, added_at) VALUES (?, ?, ?)",
                         (user_id, message.from_user.id, datetime.now().isoformat()))
        await db.commit()
    await state.clear()
    await message.answer(f"✅ Пользователь <code>{user_id}</code> добавлен в администраторы.", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, "👑 Вы назначены администратором бота!")
    except:
        pass


@dp.message(PanelStates.remove_admin)
async def remove_admin_process(message: Message, state: FSMContext):
    if not await is_admin_user(message.from_user.id):
        return
    if not message.text.strip().lstrip("-").isdigit():
        await message.answer("⚠️ Формат: <code>ID</code>", reply_markup=cancel_keyboard())
        return
    user_id = int(message.text.strip())
    if user_id == ADMIN_ID:
        await message.answer("⚠️ Владельца нельзя удалить.", reply_markup=cancel_keyboard())
        return
    async with aiosqlite.connect(DB_NAME) as db:
        cursor = await db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,))
        if not await cursor.fetchone():
            await message.answer("⚠️ Не является администратором.", reply_markup=cancel_keyboard())
            return
        await db.execute("DELETE FROM admins WHERE user_id = ?", (user_id,))
        await db.commit()
    await state.clear()
    await message.answer(f"✅ Пользователь <code>{user_id}</code> удалён из администраторов.", reply_markup=panel_keyboard())
    try:
        await bot.send_message(user_id, "❌ Вы лишены прав администратора.")
    except:
        pass


# ==================== ПОЛЬЗОВАТЕЛЬСКИЙ ЗАПРОС ====================

@dp.message()
async def user_message_handler(message: Message):
    if await is_admin_user(message.from_user.id) and message.text and message.text.startswith("/"):
        return

    if not message.text:
        await message.answer("Отправьте текстовый вопрос.")
        return

    user = await get_user(message.from_user.id, message.from_user.username or "")
    if is_banned(user):
        await message.answer("🚫 Вы заблокированы администратором.")
        return

    limit = daily_limit(user)
    if user[2] >= limit:
        await message.answer(
            "❌ <b>Запросы закончились.</b>\n\n"
            "🎁 Заберите ежедневный бонус, выполните задание "
            "или крутите мини-игру в меню — там можно получить "
            "дополнительные запросы. Либо оформите Premium в «Магазине».",
            reply_markup=main_keyboard()
        )
        return

    status_msg = await message.answer("🧠 Анализирую ваш запрос...")

    try:
        answer = await ask_tooken(message.text)
        await update_user(
            message.from_user.id,
            "UPDATE users SET requests_used = requests_used + 1 WHERE user_id = ?",
            (message.from_user.id,)
        )
        await status_msg.edit_text(answer)

        async with aiosqlite.connect(DB_NAME) as db:
            await db.execute(
                "INSERT INTO logs (user_id, username, question, answer, timestamp) VALUES (?, ?, ?, ?, ?)",
                (message.from_user.id, message.from_user.username or "", message.text, answer, datetime.now().isoformat())
            )
            await db.commit()

    except Exception as e:
        error_text = f"❌ Ошибка: {str(e)}"
        await status_msg.edit_text(error_text)
        print(f"[ERROR] {traceback.format_exc()}")
        for admin_id in await all_admin_ids():
            try:
                await bot.send_message(
                    admin_id,
                    f"⚠️ Ошибка у пользователя {message.from_user.id}\n\n"
                    f"Текст: {message.text[:200]}\n\n"
                    f"Ошибка: {traceback.format_exc()[:500]}"
                )
            except:
                pass


# ==================== ЗАПУСК ====================

async def daily_push_loop():
    while True:
        now = datetime.now()
        next_run = (now + timedelta(days=1)).replace(hour=0, minute=5, second=0, microsecond=0)
        await asyncio.sleep((next_run - now).total_seconds())
        for user_id in await all_user_ids():
            try:
                await bot.send_message(
                    user_id,
                    "🌅 <b>Новый день начался!</b>\n\n"
                    "Ваш дневной лимит запросов обновлён.\n"
                    f"🎁 Заберите бесплатный бонус: +{DAILY_BONUS_AMOUNT} запрос!",
                    reply_markup=main_keyboard()
                )
            except:
                pass
            await asyncio.sleep(0.05)


async def main():
    if not BOT_TOKEN or not TOOKEN_API_KEY:
        raise RuntimeError("Заполните BOT_TOKEN и TOOKEN_API_KEY в файле .env")

    await init_db()
    print("ChatGPT Bot started")
    print(f"Админ-панель: /panel (ADMIN_ID={ADMIN_ID})")
    print(f"Канал-задание: {SECOND_CHANNEL_USERNAME} (награда +{TASK_CHANNEL_REWARD})")

    asyncio.create_task(daily_push_loop())
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
