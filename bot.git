# bot
import asyncio
import sqlite3
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, F
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import StatesGroup, State
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.client.default import DefaultBotProperties

# ================== ВАШИ ДАННЫЕ ==================
BOT_TOKEN = "7625699282:AAF-U7kxGebGo2F8SjPd8BxYB6UJlL4PprI"
ADMIN_ID = 1195437196
GROUP_ID = -1001786690188
# ==================================================

bot = Bot(BOT_TOKEN, default=DefaultBotProperties(parse_mode="HTML"))
dp = Dispatcher(storage=MemoryStorage())

# ================== СОСТОЯНИЯ ==================
class AdState(StatesGroup):
    collecting = State()
    preview = State()
    edit_text = State()
    reject_reason = State()

# ================== БАЗА ДАННЫХ ==================
conn = sqlite3.connect("ads.db")
cursor = conn.cursor()
cursor.execute("""CREATE TABLE IF NOT EXISTS ads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    photo TEXT,
    text TEXT,
    status TEXT,
    admin_msg_id INTEGER,
    created_at TEXT
)""")
conn.commit()

# ================== ШАБЛОН ==================
TEMPLATE = (
    "Модель:\n"
    "Память:\n"
    "Состояние:\n"
    "Цена:\n"
    "Город:\n"
    "Контакт:"
)

# ================== КНОПКИ ==================
def user_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Разместить объявление", callback_data="new_ad")]
    ])

def preview_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Подтвердить", callback_data="confirm_ad")],
        [InlineKeyboardButton(text="✏ Изменить текст", callback_data="edit_text")]
    ])

def edit_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✏ Изменить текст", callback_data="edit_text")]
    ])

def admin_kb(ad_id):
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="✅ Одобрить", callback_data=f"approve:{ad_id}"),
            InlineKeyboardButton(text="❌ Отклонить", callback_data=f"reject:{ad_id}")
        ]
    ])

# ================== /start ==================
@dp.message(F.text == "/start")
async def start(msg: Message):
    await msg.answer("Привет! 👋\nЧтобы разместить объявление о телефоне, нажми кнопку ниже:", reply_markup=user_kb())

# ================== Новое объявление ==================
@dp.callback_query(F.data == "new_ad")
async def new_ad(call: CallbackQuery, state: FSMContext):
    cursor.execute("SELECT * FROM ads WHERE user_id=? AND status='pending'", (call.from_user.id,))
    if cursor.fetchone():
        await call.message.answer("❗ У вас уже есть объявление на модерации.")
        await call.answer()
        return

    await state.set_state(AdState.collecting)
    await state.update_data(photo=None, text=None)
    await call.message.answer(
        "📸 Отправьте фото телефона и подпишите его текстом.\n"
        "Можно сначала фото, потом текст или сразу вместе.\n\n"
        "📝 Шаблон для удобства:\n" + TEMPLATE
    )
    await call.answer()

# ================== Сбор фото и текста ==================
@dp.message(AdState.collecting)
async def collect(msg: Message, state: FSMContext):
    data = await state.get_data()
    photo = data.get("photo")
    text = data.get("text")

    if msg.photo:
        photo = msg.photo[-1].file_id
        await state.update_data(photo=photo)
        if msg.caption:
            text = msg.caption
            await state.update_data(text=text)
    elif msg.text:
        text = msg.text
        await state.update_data(text=text)
    else:
        await msg.answer("❌ Пришлите фото или текст")
        return

    if not photo:
        await msg.answer("📸 Фото получено, теперь пришлите текст ✍")
        return
    if not text:
        await msg.answer("✍ Текст получен, теперь пришлите фото 📸")
        return

    # Предварительный просмотр
    await state.set_state(AdState.preview)
    dt = datetime.now().strftime("%d.%m.%Y %H:%M")
    preview_caption = f"━━━━━━━━━━━━━━\n{text}\n\n📅 {dt}\n━━━━━━━━━━━━━━"
    await msg.answer_photo(photo=photo, caption=preview_caption, reply_markup=preview_kb())

# ================== Подтверждение объявления ==================
@dp.callback_query(F.data == "confirm_ad")
async def confirm_ad(call: CallbackQuery, state: FSMContext):
    data = await state.get_data()
    photo = data.get("photo")
    text = data.get("text")

    cursor.execute(
        "INSERT INTO ads (user_id, photo, text, status, created_at) VALUES (?, ?, ?, ?, ?)",
        (call.from_user.id, photo, text, "pending", datetime.now().isoformat())
    )
    conn.commit()
    ad_id = cursor.lastrowid

    admin_message = await bot.send_photo(ADMIN_ID, photo, caption=text, reply_markup=admin_kb(ad_id))
    cursor.execute("UPDATE ads SET admin_msg_id=? WHERE id=?", (admin_message.message_id, ad_id))
    conn.commit()
    await state.clear()
    await call.message.answer("⏳ Ваше объявление отправлено на проверку модератору")
    await call.answer()

# ================== Изменить текст ==================
@dp.callback_query(F.data == "edit_text")
async def edit_text(call: CallbackQuery, state: FSMContext):
    await state.set_state(AdState.edit_text)
    await call.message.answer("✏ Отправьте новый текст для вашего объявления")
    await call.answer()

@dp.message(AdState.edit_text)
async def save_text(msg: Message, state: FSMContext):
    await state.update_data(text=msg.text)
    await state.set_state(AdState.preview)
    data = await state.get_data()
    photo = data.get("photo")
    preview_caption = f"━━━━━━━━━━━━━━\n{msg.text}\n\n📅 {datetime.now().strftime('%d.%m.%Y %H:%M')}\n━━━━━━━━━━━━━━"
    await msg.answer_photo(photo=photo, caption=preview_caption, reply_markup=preview_kb())

# ================== Одобрение и публикация с красивым оформлением ==================
@dp.callback_query(F.data.startswith("approve:"))
async def approve(call: CallbackQuery):
    if call.from_user.id != ADMIN_ID:
        return

    ad_id = int(call.data.split(":")[1])
    cursor.execute("SELECT * FROM ads WHERE id=?", (ad_id,))
    ad = cursor.fetchone()
    if not ad:
        await call.answer("❌ Объявление не найдено")
        return

    user_id, photo, text, _, admin_msg_id, created_at = ad[1], ad[2], ad[3], ad[4], ad[5], ad[6]
    dt = datetime.fromisoformat(created_at).strftime("%d.%m.%Y %H:%M")

    # Форматирование текста с эмодзи и мобильной адаптацией
    lines = text.split("\n")
    formatted_lines = []
    for line in lines:
        if "Модель" in line: formatted_lines.append(f"📱 {line}")
        elif "Память" in line: formatted_lines.append(f"💾 {line}")
        elif "Состояние" in line: formatted_lines.append(f"🔋 {line}")
        elif "Цена" in line: formatted_lines.append(f"💰 {line}")
        elif "Город" in line: formatted_lines.append(f"📍 {line}")
        elif "Контакт" in line: formatted_lines.append(f"📞 {line}")
        else:
            formatted_lines.append(line)
    formatted_text = "\n".join(formatted_lines)

    final_caption = (
        "━━━━━━━━━━━━━━\n"
        f"{formatted_text}\n\n"
        f"📌 ID: {ad_id} | 📅 {dt}\n━━━━━━━━━━━━━━"
    )

    # Кнопка для связи с пользователем
    contact_kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💬 Связаться с продавцом", url=f"tg://user?id={user_id}")]
    ])

    await bot.send_photo(GROUP_ID, photo, caption=final_caption, reply_markup=contact_kb)
    await bot.send_message(user_id, f"✅ Ваше объявление опубликовано!\n📌 ID: {ad_id}")

    await bot.delete_message(chat_id=ADMIN_ID, message_id=admin_msg_id)
    cursor.execute("UPDATE ads SET status='approved' WHERE id=?", (ad_id,))
    conn.commit()
    await call.answer("Объявление опубликовано")

# ================== Отклонение ==================
@dp.callback_query(F.data.startswith("reject:"))
async def reject(call: CallbackQuery, state: FSMContext):
    if call.from_user.id != ADMIN_ID:
        return
    ad_id = int(call.data.split(":")[1])
    await state.set_state(AdState.reject_reason)
    await state.update_data(ad_id=ad_id)
    await call.message.answer("❌ Напишите причину отклонения объявления")
    await call.answer()

@dp.message(AdState.reject_reason)
async def reject_reason(msg: Message, state: FSMContext):
    data = await state.get_data()
    ad_id = data["ad_id"]
    cursor.execute("SELECT * FROM ads WHERE id=?", (ad_id,))
    ad = cursor.fetchone()
    if not ad:
        await msg.answer("❌ Объявление не найдено")
        return

    user_id, admin_msg_id = ad[1], ad[5]
    await bot.send_message(user_id, f"❌ Ваше объявление отклонено\nПричина:\n{msg.text}")
    await bot.delete_message(chat_id=ADMIN_ID, message_id=admin_msg_id)
    cursor.execute("UPDATE ads SET status='rejected' WHERE id=?", (ad_id,))
    conn.commit()
    await state.clear()
    await msg.answer("Отказ отправлен пользователю")

# ================== Поиск объявления по ID ==================
@dp.message(F.text.startswith("/find"))
async def find_ad_by_id(msg: Message):
    if msg.from_user.id != ADMIN_ID:
        await msg.answer("❌ У вас нет доступа к этой команде")
        return

    try:
        parts = msg.text.split()
        ad_id = int(parts[1])
    except (IndexError, ValueError):
        await msg.answer("❌ Использование: /find <ID объявления>\nПример: /find 12")
        return

    cursor.execute("SELECT * FROM ads WHERE id=?", (ad_id,))
    ad = cursor.fetchone()
    if not ad:
        await msg.answer("❌ Объявление с таким ID не найдено")
        return

    user_id, photo, text, status, admin_msg_id, created_at = ad[1], ad[2], ad[3], ad[4], ad[5], ad[6]
    dt = datetime.fromisoformat(created_at).strftime("%d.%m.%Y %H:%M")
    caption = f"━━━━━━━━━━━━━━\n{text}\n\n📌 ID: {ad_id} | Статус: {status} | 📅 {dt} | Пользователь: {user_id}\n━━━━━━━━━━━━━━"
    reply_kb = admin_kb(ad_id) if status == "pending" else None
    await msg.answer_photo(photo=photo, caption=caption, reply_markup=reply_kb)

# ================== Авто-удаление старых объявлений ==================
async def cleanup_old_ads():
    while True:
        cutoff = datetime.now() - timedelta(days=7)
        cursor.execute("SELECT id, admin_msg_id FROM ads WHERE created_at<? AND status='pending'", (cutoff.isoformat(),))
        for ad_id, admin_msg_id in cursor.fetchall():
            try:
                await bot.delete_message(chat_id=ADMIN_ID, message_id=admin_msg_id)
            except:
                pass
            cursor.execute("UPDATE ads SET status='expired' WHERE id=?", (ad_id,))
            conn.commit()
        await asyncio.sleep(3600)

# ================== Запуск ==================
async def main():
    asyncio.create_task(cleanup_old_ads())
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
