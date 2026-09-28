# 16. Собираем своего бота — от идеи до продакшена

Эта глава — практическая. Соберём **бота-регистратора для студсовета** от начала до конца, используя всё, что разобрали в предыдущих главах. Готовый проект можно взять за основу и переделать под свою задачу.

## Что будет уметь бот

- **Регистрация** — пользователь вводит ФИО, группу, email через пошаговый диалог.
- **Просмотр мероприятий** — список предстоящих событий с датами.
- **Запись на мероприятие** — Inline-кнопка под каждым событием.
- **Мои записи** — список мероприятий, на которые пользователь записан.
- **Напоминание** — за день до события бот присылает уведомление.
- **Админка** — создание мероприятий, рассылка, статистика.

Такой бот закрывает 90% задач студсовета и демонстрирует **все** темы курса: FSM, БД, клавиатуры, мидлвари, планировщик, деплой.

## Планирование схемы БД

Прежде чем писать код — спроектируем схему. Это сэкономит часы рефакторинга.

```sql
CREATE TABLE users (
    telegram_id   BIGINT PRIMARY KEY,
    full_name     TEXT NOT NULL,
    group_name    TEXT NOT NULL,
    email         TEXT,
    registered_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE events (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    description TEXT,
    event_date  TIMESTAMPTZ NOT NULL,
    location    TEXT,
    created_by  BIGINT REFERENCES users(telegram_id),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE registrations (
    user_id    BIGINT REFERENCES users(telegram_id),
    event_id   BIGINT REFERENCES events(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, event_id)
);

CREATE INDEX idx_events_date ON events(event_date);
CREATE INDEX idx_registrations_event ON registrations(event_id);
```

Три ключевые идеи:

- **Составной PK** в `registrations` не даёт записаться дважды.
- **`ON DELETE CASCADE`** — удалили мероприятие, регистрации ушли с ним.
- **Индексы** на дату и `event_id` — ускорят частые запросы.

## Структура проекта

Не пихаем всё в один файл. Разделяем по смыслу.

```text
student-bot/
├── aiogram_run.py          # точка входа
├── config.py               # настройки (pydantic-settings)
├── database.py             # пул asyncpg
├── middlewares.py          # БД, антифлуд
├── keyboards.py            # все клавиатуры
├── utils.py                # хелперы
├── repositories/
│   ├── __init__.py
│   ├── users.py
│   ├── events.py
│   └── registrations.py
├── routers/
│   ├── __init__.py
│   ├── start.py            # /start, /help
│   ├── registration.py     # FSM регистрации
│   ├── events.py           # список, запись, мои записи
│   └── admin.py            # админские команды
├── scheduler.py            # напоминания
├── .env                    # секреты (не в git!)
├── .env.example            # шаблон
├── requirements.txt
└── Dockerfile
```

Правило: **один роутер — одна область ответственности**. Так проще искать код и править.

## Шаг 1. Конфиг

Используем `pydantic-settings`, чтобы переменные окружения читались автоматически и проверялись на старте.

```python
# config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    bot_token: str
    database_url: str
    admin_ids: list[int] = []
    redis_url: str = "redis://localhost:6379/0"
    timezone: str = "Europe/Moscow"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )


settings = Settings()
```

Файл `.env`:

```env
BOT_TOKEN=123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw
DATABASE_URL=postgresql://bot:secret@localhost:5432/studentbot
ADMIN_IDS=123456789,987654321
REDIS_URL=redis://localhost:6379/0
```

И `.env.example` для репозитория — без реальных значений.

## Шаг 2. База данных

Пул соединений создаётся один раз при старте, живёт до остановки.

```python
# database.py
import asyncpg


class Database:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool: asyncpg.Pool | None = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(self.dsn, min_size=2, max_size=10)

    async def close(self):
        if self.pool:
            await self.pool.close()

    async def execute(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.execute(query, *args)

    async def fetch(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetch(query, *args)

    async def fetchrow(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetchrow(query, *args)

    async def fetchval(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetchval(query, *args)
```

## Шаг 3. Репозитории

Каждая сущность — свой файл. Хендлеры не знают про SQL, только про методы репозитория.

```python
# repositories/users.py
class UserRepository:
    def __init__(self, db):
        self.db = db

    async def get(self, telegram_id: int):
        return await self.db.fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            telegram_id,
        )

    async def create(self, telegram_id: int, full_name: str, group: str, email: str | None):
        await self.db.execute(
            """
            INSERT INTO users (telegram_id, full_name, group_name, email)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (telegram_id) DO UPDATE
            SET full_name = $2, group_name = $3, email = $4
            """,
            telegram_id, full_name, group, email,
        )

    async def exists(self, telegram_id: int) -> bool:
        return await self.db.fetchval(
            "SELECT EXISTS(SELECT 1 FROM users WHERE telegram_id = $1)",
            telegram_id,
        )
```

```python
# repositories/events.py
class EventRepository:
    def __init__(self, db):
        self.db = db

    async def upcoming(self, limit: int = 10):
        return await self.db.fetch(
            """
            SELECT * FROM events
            WHERE event_date > NOW()
            ORDER BY event_date
            LIMIT $1
            """,
            limit,
        )

    async def get(self, event_id: int):
        return await self.db.fetchrow(
            "SELECT * FROM events WHERE id = $1",
            event_id,
        )

    async def create(self, title: str, description: str, event_date, location: str, created_by: int):
        return await self.db.fetchval(
            """
            INSERT INTO events (title, description, event_date, location, created_by)
            VALUES ($1, $2, $3, $4, $5)
            RETURNING id
            """,
            title, description, event_date, location, created_by,
        )
```

```python
# repositories/registrations.py
class RegistrationRepository:
    def __init__(self, db):
        self.db = db

    async def register(self, user_id: int, event_id: int) -> bool:
        try:
            await self.db.execute(
                "INSERT INTO registrations (user_id, event_id) VALUES ($1, $2)",
                user_id, event_id,
            )
            return True
        except Exception:
            return False  # уже записан

    async def for_user(self, user_id: int):
        return await self.db.fetch(
            """
            SELECT e.id, e.title, e.event_date, e.location
            FROM registrations r
            JOIN events e ON e.id = r.event_id
            WHERE r.user_id = $1
            ORDER BY e.event_date
            """,
            user_id,
        )

    async def count_for_event(self, event_id: int) -> int:
        return await self.db.fetchval(
            "SELECT COUNT(*) FROM registrations WHERE event_id = $1",
            event_id,
        )
```

## Шаг 4. Клавиатуры

Все клавиатуры — в одном файле, чтобы не искать по проекту.

```python
# keyboards.py
from aiogram.types import (
    ReplyKeyboardMarkup, KeyboardButton,
    InlineKeyboardMarkup, InlineKeyboardButton,
)
from aiogram.utils.keyboard import InlineKeyboardBuilder


def main_menu() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="📅 Мероприятия")],
            [KeyboardButton(text="📝 Мои записи"), KeyboardButton(text="👤 Профиль")],
        ],
        resize_keyboard=True,
    )


def events_list(events) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for event in events:
        builder.button(
            text=f"Записаться — {event['title'][:30]}",
            callback_data=f"register:{event['id']}",
        )
    builder.adjust(1)
    return builder.as_markup()


def event_confirm(event_id: int) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.button(text="✅ Подтвердить", callback_data=f"confirm:{event_id}")
    builder.button(text="❌ Отмена", callback_data="cancel")
    builder.adjust(2)
    return builder.as_markup()


def cancel_only() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="Отмена")]],
        resize_keyboard=True,
    )
```

## Шаг 5. Мидлвари

Подключаем БД и репозитории в каждый хендлер через `data`.

```python
# middlewares.py
from aiogram import BaseMiddleware
from repositories.users import UserRepository
from repositories.events import EventRepository
from repositories.registrations import RegistrationRepository


class DatabaseMiddleware(BaseMiddleware):
    def __init__(self, db):
        self.db = db

    async def __call__(self, handler, event, data: dict):
        data["users"] = UserRepository(self.db)
        data["events"] = EventRepository(self.db)
        data["registrations"] = RegistrationRepository(self.db)
        return await handler(event, data)
```

## Шаг 6. FSM регистрации

Простейший пошаговый диалог: имя → группа → email.

```python
# routers/registration.py
from aiogram import Router, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message, ReplyKeyboardRemove

from keyboards import main_menu

router = Router()


class Registration(StatesGroup):
    waiting_name = State()
    waiting_group = State()
    waiting_email = State()


@router.message(Command("register"))
async def start_registration(message: Message, state: FSMContext, users):
    if await users.exists(message.from_user.id):
        await message.answer("Ты уже зарегистрирован. Используй /profile.")
        return
    await state.clear()
    await message.answer("Как тебя зовут? (ФИО полностью)")
    await state.set_state(Registration.waiting_name)


@router.message(Registration.waiting_name)
async def process_name(message: Message, state: FSMContext):
    name = message.text.strip()
    if len(name) < 3:
        await message.answer("Слишком коротко. Введи ФИО полностью.")
        return
    await state.update_data(full_name=name)
    await message.answer("Укажи группу (например, ИУ7-42Б):")
    await state.set_state(Registration.waiting_group)


@router.message(Registration.waiting_group)
async def process_group(message: Message, state: FSMContext):
    group = message.text.strip()
    await state.update_data(group_name=group)
    await message.answer("Укажи email (или напиши «нет»):")
    await state.set_state(Registration.waiting_email)


@router.message(Registration.waiting_email)
async def process_email(message: Message, state: FSMContext, users):
    email = message.text.strip()
    if email.lower() != "нет" and ("@" not in email or "." not in email):
        await message.answer("Это не похоже на email. Попробуй ещё раз или напиши «нет».")
        return

    data = await state.get_data()
    await users.create(
        telegram_id=message.from_user.id,
        full_name=data["full_name"],
        group=data["group_name"],
        email=None if email.lower() == "нет" else email,
    )
    await state.clear()
    await message.answer(
        f"✅ Готово, {data['full_name']}! Теперь ты в системе.",
        reply_markup=main_menu(),
    )
```

## Шаг 7. Роутер мероприятий

Здесь основная логика: показать список, обработать запись, показать свои записи.

```python
# routers/events.py
from aiogram import Router, F
from aiogram.types import Message, CallbackQuery

from keyboards import events_list, event_confirm

router = Router()


@router.message(F.text == "📅 Мероприятия")
async def show_events(message: Message, events, users):
    if not await users.exists(message.from_user.id):
        await message.answer("Сначала зарегистрируйся — /register")
        return

    upcoming = await events.upcoming(limit=10)
    if not upcoming:
        await message.answer("Пока нет предстоящих мероприятий.")
        return

    lines = ["<b>Предстоящие мероприятия:</b>\n"]
    for e in upcoming:
        date_str = e["event_date"].strftime("%d.%m.%Y %H:%M")
        lines.append(f"• <b>{e['title']}</b>\n  📅 {date_str}\n  📍 {e['location'] or '—'}\n")

    await message.answer("\n".join(lines), reply_markup=events_list(upcoming))


@router.callback_query(F.data.startswith("register:"))
async def on_register(callback: CallbackQuery, events):
    event_id = int(callback.data.split(":")[1])
    event = await events.get(event_id)

    if not event:
        await callback.answer("Мероприятие не найдено", show_alert=True)
        return

    date_str = event["event_date"].strftime("%d.%m.%Y %H:%M")
    await callback.message.edit_text(
        f"Записаться на <b>{event['title']}</b>?\n\n📅 {date_str}\n📍 {event['location'] or '—'}",
        reply_markup=event_confirm(event_id),
    )
    await callback.answer()


@router.callback_query(F.data.startswith("confirm:"))
async def on_confirm(callback: CallbackQuery, registrations):
    event_id = int(callback.data.split(":")[1])
    success = await registrations.register(callback.from_user.id, event_id)

    if success:
        await callback.message.edit_text("✅ Ты записан! Напомню за день до мероприятия.")
    else:
        await callback.message.edit_text("Ты уже записан на это мероприятие.")
    await callback.answer()


@router.callback_query(F.data == "cancel")
async def on_cancel(callback: CallbackQuery):
    await callback.message.delete()
    await callback.answer("Отменено")


@router.message(F.text == "📝 Мои записи")
async def my_registrations(message: Message, registrations):
    rows = await registrations.for_user(message.from_user.id)
    if not rows:
        await message.answer("Ты пока никуда не записан.")
        return

    lines = ["<b>Твои записи:</b>\n"]
    for r in rows:
        date_str = r["event_date"].strftime("%d.%m.%Y %H:%M")
        lines.append(f"• <b>{r['title']}</b> — {date_str}")
    await message.answer("\n".join(lines))
```

## Шаг 8. Роутер /start и /profile

Точка входа и просмотр профиля.

```python
# routers/start.py
from aiogram import Router
from aiogram.filters import CommandStart, Command
from aiogram.types import Message

from keyboards import main_menu

router = Router()


@router.message(CommandStart())
async def cmd_start(message: Message, users):
    if not await users.exists(message.from_user.id):
        await message.answer(
            f"Привет, {message.from_user.first_name}!\n\n"
            "Это бот студсовета. Для начала зарегистрируйся — /register"
        )
        return

    await message.answer(
        f"С возвращением, {message.from_user.first_name}!",
        reply_markup=main_menu(),
    )


@router.message(Command("help"))
async def cmd_help(message: Message):
    await message.answer(
        "<b>Что я умею:</b>\n\n"
        "/start — главное меню\n"
        "/register — регистрация\n"
        "/profile — мой профиль\n"
        "/events — список мероприятий\n"
        "/my — мои записи"
    )


@router.message(Command("profile"))
async def cmd_profile(message: Message, users):
    user = await users.get(message.from_user.id)
    if not user:
        await message.answer("Ты не зарегистрирован. Введи /register.")
        return

    text = (
        f"<b>Профиль</b>\n\n"
        f"👤 {user['full_name']}\n"
        f"🎓 {user['group_name']}\n"
        f"📧 {user['email'] or '—'}\n"
        f"📅 С нами с {user['registered_at'].strftime('%d.%m.%Y')}"
    )
    await message.answer(text)
```

## Шаг 9. Админка

Команды только для админов. Создание мероприятий через FSM.

```python
# routers/admin.py
from datetime import datetime

from aiogram import Router, F
from aiogram.filters import Command, BaseFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message

from config import settings

router = Router()


class IsAdmin(BaseFilter):
    async def __call__(self, message: Message) -> bool:
        return message.from_user.id in settings.admin_ids


router.message.filter(IsAdmin())


class EventCreation(StatesGroup):
    waiting_title = State()
    waiting_date = State()
    waiting_location = State()


@router.message(Command("admin"))
async def cmd_admin(message: Message):
    await message.answer(
        "<b>Админ-команды:</b>\n\n"
        "/new_event — создать мероприятие\n"
        "/broadcast — рассылка\n"
        "/stats — статистика"
    )


@router.message(Command("new_event"))
async def start_event_creation(message: Message, state: FSMContext):
    await state.clear()
    await message.answer("Название мероприятия:")
    await state.set_state(EventCreation.waiting_title)


@router.message(EventCreation.waiting_title)
async def process_title(message: Message, state: FSMContext):
    await state.update_data(title=message.text.strip())
    await message.answer("Дата и время (формат: 15.10.2026 18:00):")
    await state.set_state(EventCreation.waiting_date)


@router.message(EventCreation.waiting_date)
async def process_date(message: Message, state: FSMContext):
    try:
        dt = datetime.strptime(message.text.strip(), "%d.%m.%Y %H:%M")
    except ValueError:
        await message.answer("Неверный формат. Попробуй ещё раз: 15.10.2026 18:00")
        return
    await state.update_data(event_date=dt)
    await message.answer("Место проведения:")
    await state.set_state(EventCreation.waiting_location)


@router.message(EventCreation.waiting_location)
async def process_location(message: Message, state: FSMContext, events):
    data = await state.get_data()
    event_id = await events.create(
        title=data["title"],
        description="",
        event_date=data["event_date"],
        location=message.text.strip(),
        created_by=message.from_user.id,
    )
    await state.clear()
    await message.answer(f"✅ Мероприятие создано (ID: {event_id})")


@router.message(Command("stats"))
async def cmd_stats(message: Message, users, events):
    users_count = await users.count() if hasattr(users, "count") else "—"
    upcoming = await events.upcoming(limit=100)
    await message.answer(
        f"<b>Статистика</b>\n\n"
        f"👥 Пользователей: {users_count}\n"
        f"📅 Предстоящих мероприятий: {len(upcoming)}"
    )
```

## Шаг 10. Планировщик напоминаний

Раз в день проверяем, кому нужно напомнить о событии.

```python
# scheduler.py
import logging
from datetime import datetime, timedelta

from apscheduler.schedulers.asyncio import AsyncIOScheduler


async def send_reminders(bot, db):
    """Отправляет напоминания за день до мероприятия."""
    tomorrow_start = datetime.now().replace(hour=0, minute=0, second=0) + timedelta(days=1)
    tomorrow_end = tomorrow_start + timedelta(days=1)

    events = await db.fetch(
        """
        SELECT id, title, event_date, location
        FROM events
        WHERE event_date BETWEEN $1 AND $2
        """,
        tomorrow_start, tomorrow_end,
    )

    for event in events:
        users = await db.fetch(
            "SELECT user_id FROM registrations WHERE event_id = $1",
            event["id"],
        )
        date_str = event["event_date"].strftime("%H:%M")
        text = (
            f"🔔 Напоминание!\n\n"
            f"Завтра в {date_str} — <b>{event['title']}</b>\n"
            f"📍 {event['location'] or '—'}"
        )
        for user in users:
            try:
                await bot.send_message(user["user_id"], text)
            except Exception as e:
                logging.warning(f"Не удалось отправить {user['user_id']}: {e}")


def setup_scheduler(bot, db) -> AsyncIOScheduler:
    scheduler = AsyncIOScheduler(timezone="Europe/Moscow")
    scheduler.add_job(
        send_reminders,
        "cron",
        hour=10, minute=0,
        args=(bot, db),
        id="daily_reminders",
        replace_existing=True,
    )
    return scheduler
```

## Шаг 11. Точка входа

Собираем всё вместе.

```python
# aiogram_run.py
import asyncio
import logging

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.redis import RedisStorage

from config import settings
from database import Database
from middlewares import DatabaseMiddleware
from scheduler import setup_scheduler
from routers import start, registration, events, admin


async def main():
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )

    bot = Bot(
        token=settings.bot_token,
        default=DefaultBotProperties(parse_mode=ParseMode.HTML),
    )
    storage = RedisStorage.from_url(settings.redis_url)
    dp = Dispatcher(storage=storage)

    db = Database(settings.database_url)
    await db.connect()

    dp.update.outer_middleware(DatabaseMiddleware(db))

    dp.include_router(admin.router)
    dp.include_router(start.router)
    dp.include_router(registration.router)
    dp.include_router(events.router)

    scheduler = setup_scheduler(bot, db)
    scheduler.start()

    await bot.delete_webhook(drop_pending_updates=True)

    try:
        await dp.start_polling(bot)
    finally:
        scheduler.shutdown(wait=False)
        await db.close()


if __name__ == "__main__":
    asyncio.run(main())
```

## Шаг 12. Требования и Docker

```
# requirements.txt
aiogram==3.13.1
asyncpg==0.29.0
pydantic-settings==2.5.2
apscheduler==3.10.4
redis==5.0.8
```

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "aiogram_run.py"]
```

```yaml
# docker-compose.yml
services:
  bot:
    build: .
    restart: always
    env_file: .env
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: bot
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: studentbot
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: always

volumes:
  pgdata:
```

## Шаг 13. Проверка

Запустите локально:

```bash
docker compose up -d db redis
python aiogram_run.py
```

Сценарии для проверки:

1. **`/start`** — бот приветствует, предлагает зарегистрироваться.
2. **`/register`** — пройти весь диалог: имя → группа → email.
3. **Регистрация с ошибками** — ввести короткое имя, email без `@`. Бот должен вернуть на тот же шаг.
4. **`/profile`** — данные отображаются.
5. **`Мероприятия`** — если их нет, бот сообщает.
6. **`/new_event`** (админ) — создать мероприятие на завтра.
7. **Записаться** — нажать кнопку, подтвердить.
8. **Повторная запись** — бот должен сказать «уже записан».
9. **`Мои записи`** — событие отображается.
10. **Напоминание** — дождаться 10:00 или временно поменять время в планировщике.

## Что можно добавить дальше

- **Отмена записи** — кнопка под «Мои записи».
- **Экспорт в CSV** для админа — список записавшихся на мероприятие.
- **Напоминание за час** — второй job в планировщике.
- **Категории мероприятий** — добавить поле `category` в БД.
- **Ограничение мест** — если на событие 30 мест, а записалось 30 — не пускать.
- **Обратная связь после события** — попросить оценку через день после завершения.

## Итог

Мы собрали полноценного бота, который демонстрирует:

- **FSM** — пошаговая регистрация и создание событий.
- **Репозитории** — чистое разделение SQL и логики.
- **Мидлвари** — инъекция зависимостей.
- **Планировщик** — напоминания.
- **Redis** — FSM переживает рестарт.
- **Безопасность** — токены в `.env`, параметризованные запросы, проверка прав.
- **Docker** — воспроизводимый деплой.

Этот скелет можно расширять: добавлять новые команды, сущности, сценарии. Главное — держать структуру и не сваливать всё в один файл.

## Совет

Перед каждым добавлением новой фичи задайте два вопроса:

1. **В какой роутер это положить?** Если ни в один — возможно, нужен новый.
2. **Нужны ли новые таблицы в БД?** Если да — сначала миграция, потом код.

И ещё: **тестируйте сценарии вручную до деплоя**. FSM — самая хрупкая часть: любой забытый `state.clear()` или неверный фильтр — и пользователь застревает в диалоге, не понимая, что делать.
