<div align="center">
  <img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/0aadcaff-e2d8-4b83-b7d5-5f2d0af6eed7" />
</div>

# Руководство по Telegram-ботам на aiogram

Полный курс по созданию Telegram-ботов на Python с использованием aiogram: от первого бота до продакшена.

## О курсе

Курс рассчитан на тех, кто уже знаком с Python, но никогда не писал ботов. Мы идём от базы к продвинутым темам: разбираем, как устроен Telegram Bot API, как работает aiogram, и доходим до деплоя, тестирования и безопасности.

В каждом разделе — теория, примеры кода и практические советы.

## Содержание

### Основы

1. [Что такое Telegram-бот и как он работает](docs/01-what-is-a-bot.md)
   — Bot API, Update, Polling vs Webhook, BotFather и токен.
2. [Асинхронность и asyncio](docs/02-asyncio-basics.md)
   — Event loop, `async`/`await`, что блокирует loop, `asyncio.gather`.
3. [Архитектура aiogram: Bot, Dispatcher, Router](docs/03-architecture.md)
   — Три ключевых объекта, иерархия роутеров, точка входа.

### Логика бота

4. [Хендлеры и фильтры](docs/04-handlers-and-filters.md)
   — Виды событий, встроенные и кастомные фильтры, порядок проверки.
5. [Магические фильтры F](docs/05-magic-filters.md)
   — Условия в декораторе, комбинации, частые ошибки.
6. [Клавиатуры](docs/06-keyboards.md)
   — Reply vs Inline, `callback_data`, валидация.
7. [Машина состояний FSM](docs/07-fsm.md)
   — Пошаговые диалоги, `StatesGroup`, Redis как хранилище.
8. [Мидлвари](docs/08-middlewares.md)
   — Outer vs inner, антифлуд, инъекция зависимостей, проверка подписки.

### Данные и автоматизация

9. [Работа с базой данных](docs/09-database.md)
   — PostgreSQL, asyncpg, пул соединений, SQLAlchemy, миграции.
10. [Планировщик задач](docs/10-scheduler.md)
    — APScheduler, триггеры, идемпотентность, мониторинг.
11. [Деплой](docs/11-deploy.md)
    — Amvera, Vercel, VPS, Docker, systemd, секреты.

### Дополнительно

12. [Форматирование текста и отправка медиа](docs/12-formatting-and-media.md)
    — HTML-разметка, `FSInputFile`, альбомы, скачивание файлов.
13. [Тестирование ботов](docs/13-testing.md)
    — Unit-тесты, `feed_raw_update`, моки, CI/CD.
14. [Безопасность](docs/14-security.md)
    — Токен, SQL-инъекции, `callback_data`, secret_token, чек-лист.
15. [Продвинутые темы](docs/15-advanced.md)
    — Inline-режим, Mini Apps, платежи, локализация, микросервисы.

## Быстрый старт

Минимальный набор шагов, чтобы запустить первого бота.

```bash
# 1. Клонируем репозиторий
git clone https://github.com/yourname/aiogram-course.git
cd aiogram-course

# 2. Создаём виртуальное окружение
python3.12 -m venv .venv
source .venv/bin/activate  # для Windows: .venv\Scripts\activate

# 3. Ставим зависимости
pip install -r requirements.txt

# 4. Настраиваем окружение
cp .env.example .env
# Откройте .env и вставьте токен от @BotFather

# 5. Запускаем
python aiogram_run.py
```

## Что понадобится

- **Python 3.11+** (в примерах используется 3.12).
- **Токен бота** — получите у [@BotFather](https://t.me/BotFather).
- **PostgreSQL** — если планируете работать с БД (можно поднять через Docker).
- **Redis** — для FSM-хранилища и кеша в продакшене.

## Установка зависимостей

Основные зависимости:

```txt
aiogram>=3.15
pydantic-settings
asyncpg
sqlalchemy[asyncio]
alembic
apscheduler
redis
python-dotenv
```

Для разработки:

```txt
pytest
pytest-asyncio
pytest-cov
pip-audit
```

## Структура проекта

Типичная структура бота, к которой мы приходим к концу курса:

```
project/
├── aiogram_run.py         # точка входа
├── config.py              # настройки через pydantic-settings
├── .env                   # секреты (не в git)
├── .env.example           # шаблон
├── .gitignore
├── requirements.txt
├── routers/
│   ├── __init__.py
│   ├── start.py
│   ├── events.py
│   └── admin.py
├── middlewares/
│   ├── __init__.py
│   ├── database.py
│   └── throttling.py
├── filters/
│   └── is_admin.py
├── database/
│   ├── __init__.py
│   ├── models.py
│   └── repositories.py
├── utils/
└── tests/
    ├── conftest.py
    └── test_handlers.py
```

## Как читать курс

1. **Последовательно.** Главы идут от простого к сложному, каждая опирается на предыдущую.
2. **С практикой.** После каждой главы пишите код — без практики знания улетучиваются.
3. **Возвращайтесь.** Многие вещи становятся понятны только со второго прочтения, когда появится контекст.
