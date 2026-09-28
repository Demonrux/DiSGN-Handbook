# 9. Работа с базой данных

Рано или поздно бот должен что-то запоминать между запусками: пользователей, их данные, историю действий. Для этого нужна база данных.

## Зачем база

Файлы вроде JSON или CSV подходят для крошечных задач, но плохо масштабируются: сложно искать, обновлять и обеспечивать целостность. Полноценная база данных даёт транзакции, индексы, связи между таблицами и защиту от потери данных.

Для Telegram-ботов самый популярный выбор — **PostgreSQL**. Причины: бесплатно, надёжно, огромное сообщество, хорошо работает с асинхронным доступом.

Альтернативы:

| БД | Когда подойдёт |
|---|---|
| SQLite | Прототип, один пользователь, встроенная в приложение |
| PostgreSQL | Почти всегда — продакшен, много пользователей, сложные запросы |
| MongoDB | Если данные без чёткой схемы (но для ботов редкость) |
| Redis | Не как основная БД, а как кеш или хранилище FSM |

Мы будем говорить про PostgreSQL — он закрывает 95% задач.

## Асинхронный драйвер

С обычным `psycopg2` запросы блокируют event loop. Для aiogram нужен асинхронный драйвер — **asyncpg**. Он делает запросы не блокирующими, и бот продолжает обрабатывать сообщения, пока идёт обращение к базе.

Есть два подхода:

1. **Чистый asyncpg** — быстро, минимально, но много ручной работы.
2. **SQLAlchemy 2.0 + asyncpg** — ORM, миграции, типы, но чуть сложнее в настройке.

Для учебных проектов и небольших ботов хватит asyncpg. Для всего, что планирует расти, — SQLAlchemy.

## Пул соединений

Открывать новое соединение с базой на каждый запрос — дорого. Поэтому создаётся **пул** — набор готовых соединений, которые переиспользуются.

```python
import asyncpg

class PostgresHandler:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(self.dsn)

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

**Возвращаемое значение execute()**. В отличие от fetch, execute возвращает не строки, а строку-статус вида "INSERT 0 1" или "UPDATE 3". Если нужно получить вставленную строку или id — используйте fetchrow/fetchval с RETURNING. В EventRepository.create() показан именно этот приём.

`create_pool` создаёт пул соединений. `pool.acquire()` берёт одно соединение на время запроса и возвращает его обратно. Такой подход выдерживает нагрузку без постоянного открытия новых соединений.

Использование:

```python
db = PostgresHandler("postgresql://user:pass@localhost/mydb")
await db.connect()

# Один запрос
await db.execute("INSERT INTO users (telegram_id, full_name) VALUES ($1, $2)", user_id, name)

# Несколько строк
rows = await db.fetch("SELECT * FROM users WHERE group_name = $1", "ИУ7-42")
for row in rows:
    print(row["telegram_id"], row["full_name"])

# Одна строка
user = await db.fetchrow("SELECT * FROM users WHERE telegram_id = $1", user_id)
```

**Важно:** asyncpg использует **позиционные параметры** `$1`, `$2`, а не `?` как в SQLite. Порядок аргументов важен.

### Подключение через мидлварь

Чтобы не таскать `db` руками, подключим её через мидлварь (глава 8):

```python
class DatabaseMiddleware(BaseMiddleware):
    def __init__(self, db):
        self.db = db

    async def __call__(self, handler, event, data: dict):
        data["db"] = self.db
        return await handler(event, data)

# В main:
dp.update.outer_middleware(DatabaseMiddleware(db))
```

И в любом хендлере:

```python
@router.message(CommandStart())
async def cmd_start(message: Message, db):
    await db.execute(
        "INSERT INTO users (telegram_id, full_name) VALUES ($1, $2) ON CONFLICT DO NOTHING",
        message.from_user.id,
        message.from_user.full_name,
    )
    await message.answer("Привет!")
```

## Схема

Продуманная схема — залог того, что бот будет легко расширяться. Классический пример — три таблицы для бота-регистратора.

```sql
CREATE TABLE users (
    telegram_id BIGINT PRIMARY KEY,
    full_name   TEXT NOT NULL,
    group_name  TEXT,
    email       TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE events (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    description TEXT,
    event_date  TIMESTAMPTZ NOT NULL,
    location    TEXT,
    registration_open BOOLEAN DEFAULT TRUE
);

CREATE TABLE registrations (
    user_id  BIGINT REFERENCES users(telegram_id),
    event_id INT REFERENCES events(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, event_id)
);
```

Три ключевых идеи:

- **Primary key** однозначно идентифицирует строку. `users.telegram_id` — потому что у нас уже есть уникальный ID от Telegram, нет смысла плодить свой.
- **Foreign key** связывает таблицы. `registrations.user_id` ссылается на `users.telegram_id`, `registrations.event_id` — на `events.id`. База не даст вставить регистрацию для несуществующего пользователя или мероприятия.
- **Составной primary key** в `registrations` не даёт записаться на одно мероприятие дважды. Одна пара `(user_id, event_id)` — одна строка.

### Типы данных

Пара слов про типы:

- `BIGINT` для `telegram_id` — Telegram ID может быть больше 2^31, обычный `INTEGER` не влезет.
- `TIMESTAMPTZ` вместо `TIMESTAMP` — хранит дату со смещением. Всегда используйте `TIMESTAMPTZ`, иначе получите сюрпризы при смене часового пояса.
- `TEXT` вместо `VARCHAR(n)` — в PostgreSQL `TEXT` не медленнее, а ограничение длины всё равно почти никогда не нужно на уровне БД.
- `SERIAL` — автоинкремент. Для новых проектов лучше `BIGSERIAL` — не исчерпается.

## Основные запросы

Разберём частые паттерны на примере бота-регистратора.

**Создать или обновить пользователя (upsert):**

```python
await db.execute(
    """
    INSERT INTO users (telegram_id, full_name, group_name)
    VALUES ($1, $2, $3)
    ON CONFLICT (telegram_id)
    DO UPDATE SET full_name = $2, group_name = $3
    """,
    user_id, full_name, group_name,
)
```

`ON CONFLICT` — это PostgreSQL-специфичный синтаксис. Он позволяет одной командой сделать и вставку, и обновление.

**Получить пользователя:**

```python
user = await db.fetchrow("SELECT * FROM users WHERE telegram_id = $1", user_id)
if user is None:
    # пользователя нет
    ...
```

**Список мероприятий, на которые открыта регистрация:**

```python
events = await db.fetch(
    "SELECT * FROM events WHERE registration_open = TRUE ORDER BY event_date"
)
```

**Записать пользователя на мероприятие:**

```python
try:
    await db.execute(
        "INSERT INTO registrations (user_id, event_id) VALUES ($1, $2)",
        user_id, event_id,
    )
except asyncpg.UniqueViolationError:
    # уже записан
    ...
```

`UniqueViolationError` — исключение, которое бросает asyncpg при нарушении `PRIMARY KEY` или `UNIQUE`. Ловить его — нормальная практика, это дешевле, чем делать `SELECT` перед `INSERT`.

**Список записей пользователя с деталями мероприятий (JOIN):**

```python
rows = await db.fetch(
    """
    SELECT e.title, e.event_date, r.created_at
    FROM registrations r
    JOIN events e ON e.id = r.event_id
    WHERE r.user_id = $1
    ORDER BY e.event_date
    """,
    user_id,
)
```

JOIN — мощный инструмент. Если данные нужны из нескольких таблиц сразу, не бойтесь писать JOIN: это быстрее, чем делать несколько запросов и склеивать в Python.

## Транзакции

Если несколько запросов должны выполниться «всё или ничего», используйте транзакцию.

```python
async with db.pool.acquire() as conn:
    async with conn.transaction():
        await conn.execute("UPDATE users SET ... WHERE ...", ...)
        await conn.execute("INSERT INTO ... VALUES ...", ...)
```

Если внутри блока `transaction()` произойдёт исключение — все изменения откатятся. Это критично для операций вроде «списать деньги и записать в лог»: либо оба действия, либо ни одного.

## SQLAlchemy async (для больших проектов)

Когда таблиц становится много, а запросы обрастают условиями, чистый SQL превращается в кашу. Тогда на помощь приходит SQLAlchemy 2.0 с асинхронной поддержкой.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    telegram_id: Mapped[int] = mapped_column(primary_key=True)
    full_name: Mapped[str]
    group_name: Mapped[str | None]
    email: Mapped[str | None]

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

Использование в хендлере:

```python
from sqlalchemy import select

@router.message(Command("me"))
async def cmd_me(message: Message):
    async with async_session() as session:
        user = await session.get(User, message.from_user.id)
        if user:
            await message.answer(f"{user.full_name}, группа {user.group_name}")
        else:
            await message.answer("Ты не зарегистрирован")
```

Плюсы SQLAlchemy:

- **Типизация.** Поля описуются в классах, IDE подсказывает.
- **Безопасность.** Запросы строятся через API, SQL-инъекции исключены по умолчанию.
- **Связи.** `relationship()` между моделями.
- **Миграции.** Alembic — стандарт индустрии.

Минусы: чуть больше кода на старте, нужна настройка. Для бота с 2–3 таблицами это перебор, для бота с 10+ — маст-хэв.

## Миграции через Alembic

Схема БД меняется со временем: добавляете поля, таблицы, индексы. Вручную править базу — плохая идея (забудете применить на проде). Alembic решает эту задачу через версионированные миграции.

Установка и инициализация:

```bash
pip install alembic
alembic init migrations
```

В `alembic.ini` указываете URL базы, в `migrations/env.py` подключаете `Base.metadata`.

Создание миграции по изменениям моделей:

```bash
alembic revision --autogenerate -m "add email to users"
```

Применение:

```bash
alembic upgrade head
```

Откат:

```bash
alembic downgrade -1
```

Alembic сравнивает модели с текущей схемой и генерирует SQL для приведения в соответствие. Это не заменяет ревью — сгенерированные миграции нужно глазами проверить, особенно при удалении полей.

## Индексы

Индекс ускоряет поиск, но замедляет запись и занимает место. Индексируйте то, по чему часто ищете.

```sql
CREATE INDEX idx_users_group ON users(group_name);
CREATE INDEX idx_events_date ON events(event_date);
CREATE INDEX idx_registrations_event ON registrations(event_id);
```

`PRIMARY KEY` и `UNIQUE` автоматически создают индексы — отдельно их создавать не нужно.

Проверить, использует ли запрос индекс, можно через `EXPLAIN ANALYZE`:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE group_name = 'ИУ7-42';
```

Если видите `Seq Scan` на большой таблице — возможно, не хватает индекса.

## Обработка ошибок

Запросы к БД могут падать: сеть, блокировки, нарушение ограничений. Оборачивайте критичные места в `try/except`.

```python
import asyncpg

@router.message(Command("register"))
async def cmd_register(message: Message, db):
    try:
        await db.execute(
            "INSERT INTO registrations (user_id, event_id) VALUES ($1, $2)",
            message.from_user.id, event_id,
        )
        await message.answer("Ты записан!")
    except asyncpg.UniqueViolationError:
        await message.answer("Ты уже записан на это мероприятие")
    except asyncpg.ForeignKeyViolationError:
        await message.answer("Мероприятие не найдено")
    except Exception as e:
        logging.exception(f"DB error: {e}")
        await message.answer("Что-то пошло не так, попробуй позже")
```

Конкретные исключения (`UniqueViolationError`, `ForeignKeyViolationError`) обрабатывайте отдельно — от них зависит текст ответа пользователю. Общий `Exception` — для всего остального, с логированием.

## Подключение к main

Собираем всё вместе:

```python
async def main():
    # ... создание bot, dp ...

    db = PostgresHandler("postgresql://user:pass@localhost/mydb")
    await db.connect()

    dp.update.outer_middleware(DatabaseMiddleware(db))

    try:
        await dp.start_polling(bot)
    finally:
        await db.pool.close()  # закрыть пул при остановке
```

`finally` гарантирует, что пул закроется даже при ошибке. Иначе соединения останутся висеть в PostgreSQL до таймаута.

## Совет

Держите все SQL-запросы в одном месте — например, в классе-репозитории. Хендлер не должен знать про структуру таблиц.

```python
class UserRepository:
    def __init__(self, db):
        self.db = db

    async def get(self, telegram_id: int):
        return await self.db.fetchrow("SELECT * FROM users WHERE telegram_id = $1", telegram_id)

    async def upsert(self, telegram_id: int, full_name: str, group: str | None = None):
        await self.db.execute(
            """
            INSERT INTO users (telegram_id, full_name, group_name)
            VALUES ($1, $2, $3)
            ON CONFLICT (telegram_id) DO UPDATE
            SET full_name = $2, group_name = $3
            """,
            telegram_id, full_name, group,
        )
```

И в хендлере:

```python
@router.message(CommandStart())
async def cmd_start(message: Message, users: UserRepository):
    await users.upsert(message.from_user.id, message.from_user.full_name)
    await message.answer("Привет!")
```

Так код остаётся чистым, а SQL — в одном месте. При смене схемы правите только репозиторий, а не десяток хендлеров.
