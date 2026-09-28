# 9. Работа с базой данных

Рано или поздно бот должен что-то запоминать между запусками: пользователей, их данные, историю действий. Для этого нужна база данных.

## Зачем база

Файлы вроде JSON или CSV подходят для крошечных задач, но плохо масштабируются: сложно искать, обновлять и обеспечивать целостность. Полноценная база данных даёт транзакции, индексы, связи между таблицами и защиту от потери данных.

Для Telegram-ботов самый популярный выбор — **PostgreSQL**. Причины: бесплатно, надёжно, огромное сообщество, хорошо работает с асинхронным доступом.

## Асинхронный драйвер

С обычным `psycopg2` запросы блокируют event loop. Для aiogram нужен асинхронный драйвер — **asyncpg**. Он делает запросы не блокирующими, и бот продолжает обрабатывать сообщения, пока идёт обращение к базе.

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

    async def execute(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.execute(query, *args)

    async def fetch(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetch(query, *args)
```

`create_pool` создаёт пул соединений. `pool.acquire()` берёт одно соединение на время запроса и возвращает его обратно. Такой подход выдерживает нагрузку без постоянного открытия новых соединений.

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

Три ключевых идеи: **primary key** однозначно идентифицирует строку, **foreign key** связывает таблицы, **составной primary key** в `registrations` не даёт записаться на одно мероприятие дважды.
