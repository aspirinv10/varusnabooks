import os
import aiosqlite
from aiogram import Bot, Dispatcher, F, Router
from aiogram.types import Message, CallbackQuery, InlineKeyboardButton, InlineKeyboardMarkup
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.enums import ParseMode

# ============ НАСТРОЙКИ ============
BOT_TOKEN = ""
OWNER_ID = 8777292199  # ← Твой Telegram ID

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
router = Router()
dp.include_router(router)

# ============ БАЗА ДАННЫХ ============
DB_PATH = "books.db"
CHUNK_SIZE = 3900  # размер страницы (с запасом от лимита 4096)

async def init_db():
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS books (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                description TEXT
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS chapters (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                book_id INTEGER NOT NULL,
                chapter_num INTEGER NOT NULL,
                title TEXT NOT NULL,
                content TEXT NOT NULL,
                FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS bookmarks (
                user_id INTEGER PRIMARY KEY,
                book_id INTEGER,
                chapter_num INTEGER,
                page INTEGER DEFAULT 1
            )
        """)
        # Миграция для старых БД: добавляем поле page если его нет
        try:
            await db.execute("ALTER TABLE bookmarks ADD COLUMN page INTEGER DEFAULT 1")
        except Exception:
            pass
        await db.commit()

# ============ FSM ============
class AddBook(StatesGroup):
    title = State()
    description = State()

class AddChapter(StatesGroup):
    book_id = State()
    title = State()
    content = State()

# ============ ХЕЛПЕРЫ ============
def is_owner(user_id: int) -> bool:
    return user_id == OWNER_ID

def split_text(text: str, max_len: int = CHUNK_SIZE) -> list[str]:
    """Разбивает текст на части по max_len символов, не разрывая слова."""
    if len(text) <= max_len:
        return [text]
    
    chunks = []
    while text:
        if len(text) <= max_len:
            chunks.append(text)
            break
        # Ищем последнее пробельное место до лимита
        cut = text.rfind(" ", 0, max_len)
        if cut == -1 or cut < max_len // 2:
            # Если не нашли — режем по переносу строки
            cut = text.rfind("\n", 0, max_len)
        if cut == -1 or cut < max_len // 2:
            cut = max_len  # жёстко режем
        chunks.append(text[:cut].rstrip())
        text = text[cut:].lstrip()
    return chunks

def main_menu_kb(is_admin: bool) -> InlineKeyboardMarkup:
    buttons = [[InlineKeyboardButton(text="📚 Книги", callback_data="menu:books")]]
    if is_admin:
        buttons.append([InlineKeyboardButton(text="⚙️ Админ-панель", callback_data="admin:menu")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def back_to_menu_kb() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🏠 Главное меню", callback_data="menu:main")]
    ])

# ============ СТАРТ ============
@router.message(Command("start"))
async def cmd_start(message: Message):
    is_admin = is_owner(message.from_user.id)
    text = (
        "✨ <b>Добро пожаловать в BookReader Bot!</b> ✨\n\n"
        "📖 Здесь ты можешь читать книги по главам,\n"
        "удобно перелистывая их кнопками.\n\n"
        "👇 Выбери действие:"
    )
    await message.answer(text, reply_markup=main_menu_kb(is_admin), parse_mode=ParseMode.HTML)

# ============ ГЛАВНОЕ МЕНЮ ============
@router.callback_query(F.data == "menu:main")
async def cb_main(call: CallbackQuery):
    is_admin = is_owner(call.from_user.id)
    await call.message.edit_text(
        "✨ <b>Главное меню</b> ✨\n\n👇 Выбери действие:",
        reply_markup=main_menu_kb(is_admin), parse_mode=ParseMode.HTML
    )
    await call.answer()

@router.callback_query(F.data == "menu:books")
async def cb_books(call: CallbackQuery):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT id, title, description FROM books ORDER BY id") as cursor:
            books = await cursor.fetchall()

    if not books:
        text = "📭 <b>Пока нет ни одной книги.</b>\n\nВозвращайся позже!"
        await call.message.edit_text(text, reply_markup=back_to_menu_kb(), parse_mode=ParseMode.HTML)
        await call.answer()
        return

    kb = []
    for b in books:
        kb.append([InlineKeyboardButton(text=f"📖 {b['title']}", callback_data=f"book:{b['id']}")])
    kb.append([InlineKeyboardButton(text="🔙 Назад", callback_data="menu:main")])

    await call.message.edit_text(
        "📚 <b>Список книг</b>\n\n👇 Выбери книгу для чтения:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML
    )
    await call.answer()

# ============ ЧТЕНИЕ КНИГИ ============
@router.callback_query(F.data.startswith("book:"))
async def cb_book(call: CallbackQuery):
    book_id = int(call.data.split(":")[1])
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT title, description FROM books WHERE id=?", (book_id,)) as cur:
            book = await cur.fetchone()
        async with db.execute("SELECT COUNT(*) as cnt FROM chapters WHERE book_id=?", (book_id,)) as cur:
            cnt = (await cur.fetchone())["cnt"]

    if not book or cnt == 0:
        await call.message.edit_text(
            "📭 В этой книге пока нет глав.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔙 К списку книг", callback_data="menu:books")]
            ]), parse_mode=ParseMode.HTML
        )
        await call.answer()
        return

    text = (
        f"📖 <b>{book['title']}</b>\n\n"
        f"📝 <i>{book['description'] or 'Описание отсутствует'}</i>\n\n"
        f"📑 Глав: <b>{cnt}</b>\n\n"
        f"👇 Начни чтение:"
    )
    
    # Смотрим закладку — если есть, предлагаем продолжить
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT chapter_num, page FROM bookmarks WHERE user_id=? AND book_id=?",
            (call.from_user.id, book_id)
        ) as cur:
            bm = await cur.fetchone()

    kb = []
    if bm:
        kb.append([InlineKeyboardButton(
            text=f"▶️ Продолжить (глава {bm['chapter_num']})",
            callback_data=f"read:{book_id}:{bm['chapter_num']}:{bm['page'] or 1}"
        )])
    kb.append([InlineKeyboardButton(text="📖 Начать с начала", callback_data=f"read:{book_id}:1:1")])
    kb.append([InlineKeyboardButton(text="🔙 К списку книг", callback_data="menu:books")])

    await call.message.edit_text(text, reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
    await call.answer()

@router.callback_query(F.data.startswith("read:"))
async def cb_read(call: CallbackQuery):
    parts = call.data.split(":")
    book_id = int(parts[1])
    chapter_num = int(parts[2])
    page = int(parts[3]) if len(parts) > 3 else 1

    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT chapter_num, title, content FROM chapters WHERE book_id=? AND chapter_num=?",
            (book_id, chapter_num)
        ) as cur:
            chapter = await cur.fetchone()
        async with db.execute(
            "SELECT MAX(chapter_num) as max_ch FROM chapters WHERE book_id=?", (book_id,)
        ) as cur:
            max_ch = (await cur.fetchone())["max_ch"]

    if not chapter:
        await call.answer("❌ Глава не найдена", show_alert=True)
        return

    # Разбиваем на страницы
    chunks = split_text(chapter["content"])
    total_pages = len(chunks)
    
    # Корректируем страницу если вышла за границы
    if page < 1:
        page = 1
    if page > total_pages:
        page = total_pages
    
    current_text = chunks[page - 1]

    # Сохраняем закладку
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO bookmarks (user_id, book_id, chapter_num, page) VALUES (?, ?, ?, ?)",
            (call.from_user.id, book_id, chapter_num, page)
        )
        await db.commit()

    # Формируем сообщение
    header = f"📖 <b>Глава {chapter['chapter_num']}</b>\n🏷 <i>{chapter['title']}</i>\n\n{'─' * 20}\n\n"
    footer = f"\n\n{'─' * 20}\n📊 Глава {chapter_num}/{max_ch} · Стр. {page}/{total_pages}"
    
    # Собираем с учётом лимита 4096
    full_text = header + current_text + footer
    if len(full_text) > 4096:
        # Уменьшаем chunk если не влезает с шапкой
        available = 4096 - len(header) - len(footer) - 10
        current_text = current_text[:available]
        full_text = header + current_text + footer

    # Кнопки навигации
    kb = []
    
    # Ряд 1: страницы
    page_row = []
    if total_pages > 1:
        if page > 1:
            page_row.append(InlineKeyboardButton(text="◀️ Стр", callback_data=f"read:{book_id}:{chapter_num}:{page-1}"))
        page_row.append(InlineKeyboardButton(text=f"📄 {page}/{total_pages}", callback_data="noop"))
        if page < total_pages:
            page_row.append(InlineKeyboardButton(text="Стр ▶️", callback_data=f"read:{book_id}:{chapter_num}:{page+1}"))
    if page_row:
        kb.append(page_row)
    
    # Ряд 2: главы
    ch_row = []
    if chapter_num > 1:
        ch_row.append(InlineKeyboardButton(text="◀️ Глава", callback_data=f"read:{book_id}:{chapter_num-1}:1"))
    ch_row.append(InlineKeyboardButton(text=f"📑 {chapter_num}/{max_ch}", callback_data="noop"))
    if chapter_num < max_ch:
        ch_row.append(InlineKeyboardButton(text="Глава ▶️", callback_data=f"read:{book_id}:{chapter_num+1}:1"))
    kb.append(ch_row)
    
    # Ряд 3: меню
    kb.append([
        InlineKeyboardButton(text="📑 К главам", callback_data=f"chapters:{book_id}"),
        InlineKeyboardButton(text="🏠 Меню", callback_data="menu:main")
    ])

    try:
        await call.message.edit_text(full_text, reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
    except Exception as e:
        # Если всё равно не влезает — отправим как документ
        import io
        doc = io.BytesIO(chapter["content"].encode("utf-8"))
        doc.name = f"chapter_{chapter_num}.txt"
        await call.message.answer_document(
            document=doc,
            caption=f"📖 Глава {chapter_num}: {chapter['title']}",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=kb)
        )
        await call.message.delete()
    await call.answer()

@router.callback_query(F.data.startswith("chapters:"))
async def cb_chapters(call: CallbackQuery):
    book_id = int(call.data.split(":")[1])
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT chapter_num, title FROM chapters WHERE book_id=? ORDER BY chapter_num",
            (book_id,)
        ) as cur:
            chapters = await cur.fetchall()

    kb = []
    for ch in chapters:
        kb.append([InlineKeyboardButton(text=f"📄 Глава {ch['chapter_num']}: {ch['title'][:25]}",
                                        callback_data=f"read:{book_id}:{ch['chapter_num']}:1")])
    kb.append([InlineKeyboardButton(text="🔙 К книге", callback_data=f"book:{book_id}")])

    await call.message.edit_text(
        "📑 <b>Список глав</b>\n\n👇 Выбери главу:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML
    )
    await call.answer()

@router.callback_query(F.data == "noop")
async def cb_noop(call: CallbackQuery):
    await call.answer()

# ============ АДМИН-ПАНЕЛЬ ============
@router.callback_query(F.data == "admin:menu")
async def cb_admin_menu(call: CallbackQuery):
    if not is_owner(call.from_user.id):
        await call.answer("⛔ Доступ запрещён", show_alert=True)
        return

    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM books") as cur:
            books_cnt = (await cur.fetchone())[0]
        async with db.execute("SELECT COUNT(*) FROM chapters") as cur:
            chapters_cnt = (await cur.fetchone())[0]

    text = (
        "⚙️ <b>Админ-панель</b> ⚙️\n\n"
        "📊 <b>Статистика:</b>\n"
        f"   📚 Книг: <b>{books_cnt}</b>\n"
        f"   📄 Глав: <b>{chapters_cnt}</b>\n\n"
        "👇 Выбери действие:"
    )
    kb = [
        [InlineKeyboardButton(text="➕ Добавить книгу", callback_data="admin:addbook")],
        [InlineKeyboardButton(text="📝 Добавить главу", callback_data="admin:addchapter")],
        [InlineKeyboardButton(text="📋 Управление книгами", callback_data="admin:listbooks")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="menu:main")]
    ]
    await call.message.edit_text(text, reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
    await call.answer()

@router.callback_query(F.data == "admin:addbook")
async def cb_addbook(call: CallbackQuery, state: FSMContext):
    if not is_owner(call.from_user.id):
        await call.answer("⛔", show_alert=True)
        return
    await call.message.edit_text(
        "📚 <b>Добавление книги</b>\n\n✏️ Введи <b>название книги</b>:\n\n❌ Для отмены: /cancel",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AddBook.title)
    await call.answer()

@router.message(StateFilter(AddBook.title))
async def addbook_title(message: Message, state: FSMContext):
    if not is_owner(message.from_user.id):
        return
    await state.update_data(title=message.text.strip())
    await message.answer(
        "📝 Теперь введи <b>описание</b> книги (или отправь <code>-</code> чтобы пропустить):",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AddBook.description)

@router.message(StateFilter(AddBook.description))
async def addbook_description(message: Message, state: FSMContext):
    if not is_owner(message.from_user.id):
        return
    desc = message.text.strip()
    if desc == "-":
        desc = None
    data = await state.get_data()
    async with aiosqlite.connect(DB_PATH) as db:
        cursor = await db.execute(
            "INSERT INTO books (title, description) VALUES (?, ?)",
            (data["title"], desc)
        )
        await db.commit()
        book_id = cursor.lastrowid
    await state.clear()
    await message.answer(
        f"✅ <b>Книга добавлена!</b>\n\n📚 Название: <b>{data['title']}</b>\n🆔 ID: <code>{book_id}</code>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="⚙️ Админ-панель", callback_data="admin:menu")]
        ]),
        parse_mode=ParseMode.HTML
    )

@router.callback_query(F.data == "admin:addchapter")
async def cb_addchapter(call: CallbackQuery, state: FSMContext):
    if not is_owner(call.from_user.id):
        await call.answer("⛔", show_alert=True)
        return
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT id, title FROM books") as cur:
            books = await cur.fetchall()
    if not books:
        await call.message.edit_text(
            "⚠️ Сначала добавь хотя бы одну книгу!",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="⚙️ Админ-панель", callback_data="admin:menu")]
            ]), parse_mode=ParseMode.HTML
        )
        await call.answer()
        return
    kb = [[InlineKeyboardButton(text=f"📖 {b['title']}", callback_data=f"addch_book:{b['id']}")] for b in books]
    kb.append([InlineKeyboardButton(text="❌ Отмена", callback_data="admin:menu")])
    await call.message.edit_text(
        "📝 <b>Выбери книгу</b>, в которую хочешь добавить главу:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML
    )
    await call.answer()

@router.callback_query(F.data.startswith("addch_book:"))
async def cb_addch_book(call: CallbackQuery, state: FSMContext):
    if not is_owner(call.from_user.id):
        return
    book_id = int(call.data.split(":")[1])
    await state.update_data(book_id=book_id)
    await call.message.edit_text(
        "🏷 Введи <b>название главы</b> (например: «Глава 1. Начало»):\n\n❌ /cancel",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AddChapter.title)
    await call.answer()

@router.message(StateFilter(AddChapter.title))
async def addchapter_title(message: Message, state: FSMContext):
    if not is_owner(message.from_user.id):
        return
    await state.update_data(title=message.text.strip())
    await message.answer(
        "📄 Теперь отправь <b>текст главы</b> одним сообщением или файлом .txt:\n\n💡 Длинные тексты лучше слать файлом.\n\n❌ /cancel",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AddChapter.content)

@router.message(StateFilter(AddChapter.content), F.text)
async def addchapter_content_text(message: Message, state: FSMContext):
    if not is_owner(message.from_user.id):
        return
    await save_chapter(message, message.text, state)

@router.message(StateFilter(AddChapter.content), F.document)
async def addchapter_content_doc(message: Message, state: FSMContext):
    if not is_owner(message.from_user.id):
        return
    file = await bot.get_file(message.document.file_id)
    data = await bot.download_file(file.file_path)
    text = data.read().decode("utf-8")
    await save_chapter(message, text, state)

async def save_chapter(message: Message, content: str, state: FSMContext):
    data = await state.get_data()
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT COALESCE(MAX(chapter_num), 0) as m FROM chapters WHERE book_id=?",
            (data["book_id"],)
        ) as cur:
            next_num = (await cur.fetchone())[0] + 1
        await db.execute(
            "INSERT INTO chapters (book_id, chapter_num, title, content) VALUES (?, ?, ?, ?)",
            (data["book_id"], next_num, data["title"], content.strip())
        )
        await db.commit()
    await state.clear()
    pages = len(split_text(content))
    await message.answer(
        f"✅ <b>Глава добавлена!</b>\n\n"
        f"📄 Номер: <b>{next_num}</b>\n"
        f"🏷 Название: <b>{data['title']}</b>\n"
        f"📊 Страниц при чтении: <b>{pages}</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="⚙️ Админ-панель", callback_data="admin:menu")]
        ]),
        parse_mode=ParseMode.HTML
    )

@router.callback_query(F.data == "admin:listbooks")
async def cb_listbooks(call: CallbackQuery):
    if not is_owner(call.from_user.id):
        await call.answer("⛔", show_alert=True)
        return
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT id, title FROM books ORDER BY id") as cur:
            books = await cur.fetchall()
    if not books:
        await call.message.edit_text(
            "📭 Книг пока нет.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="⚙️ Админ-панель", callback_data="admin:menu")]
            ]), parse_mode=ParseMode.HTML
        )
        await call.answer()
        return
    kb = []
    for b in books:
        kb.append([
            InlineKeyboardButton(text=f"📖 {b['title']}", callback_data=f"admin_book:{b['id']}"),
            InlineKeyboardButton(text="🗑", callback_data=f"del_book:{b['id']}")
        ])
    kb.append([InlineKeyboardButton(text="🔙 Назад", callback_data="admin:menu")])
    await call.message.edit_text(
        "📋 <b>Управление книгами</b>\n\n🗑 — удалить книгу:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML
    )
    await call.answer()

@router.callback_query(F.data.startswith("admin_book:"))
async def cb_admin_book(call: CallbackQuery):
    if not is_owner(call.from_user.id):
        return
    book_id = int(call.data.split(":")[1])
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT id, title, description FROM books WHERE id=?", (book_id,)) as cur:
            book = await cur.fetchone()
        async with db.execute(
            "SELECT id, chapter_num, title FROM chapters WHERE book_id=? ORDER BY chapter_num",
            (book_id,)
        ) as cur:
            chapters = await cur.fetchall()
    text = (
        f"📖 <b>{book['title']}</b>\n"
        f"📝 <i>{book['description'] or '—'}</i>\n\n"
        f"📄 <b>Главы ({len(chapters)}):</b>\n"
    )
    kb = []
    for ch in chapters:
        kb.append([
            InlineKeyboardButton(text=f"#{ch['chapter_num']} {ch['title'][:20]}", callback_data="noop"),
            InlineKeyboardButton(text="🗑", callback_data=f"del_ch:{ch['id']}:{book_id}")
        ])
    kb.append([InlineKeyboardButton(text="🔙 К списку", callback_data="admin:listbooks")])
    await call.message.edit_text(text, reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
    await call.answer()

@router.callback_query(F.data.startswith("del_book:"))
async def cb_del_book(call: CallbackQuery):
    if not is_owner(call.from_user.id):
        return
    book_id = int(call.data.split(":")[1])
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM books WHERE id=?", (book_id,))
        await db.commit()
    await call.answer("🗑 Книга удалена", show_alert=True)
    await cb_listbooks(call)

@router.callback_query(F.data.startswith("del_ch:"))
async def cb_del_ch(call: CallbackQuery):
    if not is_owner(call.from_user.id):
        return
    parts = call.data.split(":")
    ch_id, book_id = int(parts[1]), int(parts[2])
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM chapters WHERE id=?", (ch_id,))
        await db.commit()
    await call.answer("🗑 Глава удалена", show_alert=True)
    await cb_admin_book(call)

@router.message(Command("cancel"))
async def cmd_cancel(message: Message, state: FSMContext):
    await state.clear()
    await message.answer("❌ Действие отменено.", reply_markup=back_to_menu_kb())

# ============ ЗАПУСК ============
async def main():
    await init_db()
    print("✅ Бот запущен!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
