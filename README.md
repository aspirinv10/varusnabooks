# 📚 BookReader Bot

Telegram-бот для чтения книг по главам с удобной навигацией.

## ✨ Возможности

### 👤 Для читателей:
- 📚 Список всех книг
- ▶️ / ◀️ Перелистывание глав и страниц
- 📑 Переход к любой главе
- 🔖 Автоматическое сохранение закладок

### 👑 Для владельца:
- ➕ Добавление книг
- 📝 Добавление глав (текстом или .txt файлом)
- 📋 Управление книгами и главами
- 📊 Статистика

## 🚀 Деплой на Railway

### 1. Форкни репозиторий
Нажми кнопку **Fork** в правом верхнем углу.

### 2. Создай бота в Telegram
- Открой [@BotFather](https://t.me/BotFather)
- Отправь `/newbot`
- Следуй инструкциям
- Скопируй токен

### 3. Узнай свой Telegram ID
- Открой [@userinfobot](https://t.me/userinfobot)
- Скопируй свой ID

### 4. Деплой на Railway
1. Зайди на [railway.app](https://railway.app)
2. Войди через GitHub
3. Нажми **New Project** → **Deploy from GitHub repo**
4. Выбери свой репозиторий
5. В настройках проекта добавь переменные окружения:
   - `BOT_TOKEN` = токен от BotFather
   - `OWNER_ID` = твой Telegram ID
6. Railway автоматически задеплоит бота

### 5. Готово!
Открой своего бота в Telegram и отправь `/start`

## 🛠 Локальный запуск

```bash
# Клонируй репозиторий
git clone https://github.com/твой-username/book-reader-bot.git
cd book-reader-bot

# Создай виртуальное окружение
python -m venv venv
source venv/bin/activate  # Linux/Mac
# или
venv\Scripts\activate  # Windows

# Установи зависимости
pip install -r requirements.txt

# Создай .env файл
cp .env.example .env
# Отредактируй .env и добавь свои токены

# Запусти бота
python bot.py