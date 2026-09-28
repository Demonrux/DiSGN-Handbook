# 8. Мидлвари

Мидлварь — это промежуточный слой, который выполняется до того, как событие попадёт в хендлер. Через мидлвари реализуются сквозные задачи: логирование, проверка прав, антифлуд, передача зависимостей.

## Идея

Представьте себе конвейер. Событие от Telegram попадает сначала в диспетчер, потом в мидлварь, потом в хендлер. На каждом шаге можно что-то сделать: записать в лог, добавить данные в контекст, остановить обработку.

Схематично:

```
Update
  │
  ▼
┌────────────────────┐
│   outer middleware │  ← до фильтров
└────────────────────┘
  │
  ▼
┌────────────────────┐
│      фильтры       │  ← подходит ли хендлеру
└────────────────────┘
  │
  ▼
┌────────────────────┐
│   middleware       │  ← после фильтров, до хендлера
└────────────────────┘
  │
  ▼
┌────────────────────┐
│      handler       │
└────────────────────┘
```

Мидлварь — это асинхронная функция, которая получает событие, словарь с данными и функцию-обработчик. Она может сделать что-то до вызова обработчика, после вызова или вообще не вызывать его.

## Простой пример: логирование

```python
from aiogram import BaseMiddleware
from aiogram.types import Message

class LoggingMiddleware(BaseMiddleware):
    async def __call__(self, handler, event: Message, data: dict):
        print(f"Пришло сообщение: {event.text}")
        return await handler(event, data)
```

Класс наследуется от `BaseMiddleware`, реализует `__call__`. Внутри — что угодно: до `await handler(...)` — код, который выполняется до обработчика, после — после.

## Подключение

Мидлварь подключается к диспетчеру или к отдельному роутеру:

```python
dp.message.middleware(LoggingMiddleware())
```

Можно подключить к `dp.message` (только к сообщениям), к `dp.callback_query` (только к нажатиям) или к `dp.update` (ко всем событиям). Если подключить к роутеру, мидлварь будет применяться только к хендлерам этого роутера.

```python
admin_router.message.middleware(LoggingMiddleware())
```

Так мидлварь сработает только внутри `admin_router`, но не затронет остальные.

## outer_middleware и middleware

Важное различие, которое часто упускают: у мидлварей есть два режима — **outer** и **inner**.

**Inner (`middleware`)** выполняется **после** фильтров. Событие уже проверено на соответствие хендлеру — мидлварь вызывается, только если фильтры совпали. Это поведение по умолчанию.

**Outer (`outer_middleware`)** выполняется **до** фильтров. Событие попадает в мидлварь независимо от того, есть ли подходящий хендлер. Это удобно для логирования, антифлуда и всего, что должно работать «всегда».

```python
# Выполнится для всех Update, даже если фильтры не совпали
dp.update.outer_middleware(LoggingMiddleware())

# Выполнится только для сообщений, которые прошли фильтры
dp.message.middleware(ThrottlingMiddleware())
```

Когда что использовать:

| Задача | Режим |
|---|---|
| Логирование всех событий | `outer` |
| Антифлуд | `outer` (отсекаем до обработки) |
| Проверка подписки | `outer` (иначе спамер проскочит) |
| Инъекция зависимостей | `outer` (нужно всем) |
| Что-то специфичное для конкретного хендлера | `inner` |

## Пример: инъекция зависимостей

Часто нужно передавать в каждый хендлер объект базы данных или что-то ещё. Вместо того чтобы делать глобальные переменные, можно использовать мидлварь.

```python
class DatabaseMiddleware(BaseMiddleware):
    def __init__(self, db):
        self.db = db

    async def __call__(self, handler, event, data: dict):
        data["db"] = self.db
        return await handler(event, data)
```

Теперь в хендлере можно принять параметр `db`:

```python
@router.message(CommandStart())
async def cmd_start(message: Message, db):
    await db.execute("INSERT INTO users (telegram_id) VALUES ($1) ON CONFLICT DO NOTHING", message.from_user.id)
    await message.answer("Привет!")
```

aiogram автоматически передаст `db` из `data` в аргументы функции, если имя совпадает. Это тот же механизм, что и с `state`, `bot` — просто теперь вы добавляете свои ключи.

Обратите внимание: `__init__` мидлвари принимает зависимости, а `__call__` кладёт их в `data`. Такой подход позволяет создавать мидлварь один раз и переиспользовать.

## Пример: антифлуд

Классическая задача — не дать пользователю спамить. Мидлварь отсекает сообщения, приходящие слишком часто.

```python
import time
from collections import defaultdict
from aiogram import BaseMiddleware
from aiogram.types import Message

class ThrottlingMiddleware(BaseMiddleware):
    def __init__(self, rate_limit: float = 1.0):
        self.rate_limit = rate_limit
        self.last_call = defaultdict(float)

    async def __call__(self, handler, event: Message, data: dict):
        user_id = event.from_user.id
        now = time.time()

        if now - self.last_call[user_id] < self.rate_limit:
            await event.answer("Слишком часто! Подожди немного.")
            return  # НЕ вызываем handler — обработка останавливается

        self.last_call[user_id] = now
        return await handler(event, data)
```

Здесь ключевой момент: если проверка не прошла, мы **не вызываем** `handler`. Обработка события останавливается, и хендлер не срабатывает.

Есть более продвинутые реализации с учётом токенов (token bucket, leaky bucket), но для базового антифлуда этого достаточно.

## Пример: проверка подписки

Классический сценарий — проверять, подписан ли пользователь на канал, прежде чем пускать его дальше.

```python
class SubscriptionMiddleware(BaseMiddleware):
    def __init__(self, channel_id: int, bot: Bot):
        self.channel_id = channel_id
        self.bot = bot

    async def __call__(self, handler, event: Message, data: dict):
        user_id = event.from_user.id
        member = await self.bot.get_chat_member(self.channel_id, user_id)

        if member.status in ("left", "kicked"):
            await event.answer("Подпишись на канал, чтобы пользоваться ботом!")
            return

        return await handler(event, data)
```

Здесь важно **не вызывать** `handler`, если проверка не прошла. В этом случае обработка события останавливается.

Обратите внимание, что мидлварь работает с `Message`. Если пользователь нажимает Inline-кнопку, событие придёт как `CallbackQuery` — и проверка не сработает. Чтобы покрыть оба типа, нужно либо зарегистрировать мидлварь и на `callback_query`, либо использовать `outer_middleware` на уровне `dp.update`.

## Пример: логирование времени обработки

Полезно для отладки — понять, какие хендлеры работают медленно.

```python
import time
import logging
from aiogram import BaseMiddleware

class TimingMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data: dict):
        start = time.perf_counter()
        try:
            result = await handler(event, data)
        finally:
            elapsed = time.perf_counter() - start
            logging.info(f"Handler took {elapsed:.3f}s")
        return result
```

`try/finally` гарантирует, что время измерится, даже если хендлер упадёт с ошибкой.

## Порядок выполнения

Если подключено несколько мидлварей, они выполняются **в порядке подключения**. Каждая мидлварь может вызывать следующую через `await handler(...)`, а может прервать цепочку.

```python
dp.message.outer_middleware(ThrottlingMiddleware())
dp.message.middleware(LoggingMiddleware())
dp.message.middleware(SubscriptionMiddleware())
```

Порядок:

```
Update
  │
  ▼
ThrottlingMiddleware (outer, до фильтров)
  │
  ▼
фильтры
  │
  ▼
LoggingMiddleware
  │
  ▼
SubscriptionMiddleware
  │
  ▼
handler
```

`outer`-мидлвари всегда выполняются до `inner`, независимо от порядка подключения.

Если одна мидлварь не вызывает `handler`, следующие мидлвари **не выполняются** — цепочка прерывается. Это важно при отладке: если событие «не доходит» до хендлера, проверьте, не отсекает ли его какая-то из ранних мидлварей.

## Обработка ошибок в мидлвари

Если мидлварь упадёт — упадёт вся обработка события. Оборачивайте потенциально опасные места в `try/except`:

```python
class SafeThrottling(BaseMiddleware):
    async def __call__(self, handler, event: Message, data: dict):
        try:
            if self.is_throttled(event.from_user.id):
                await event.answer("Подожди немного")
                return
        except Exception as e:
            logging.exception(f"Throttling error: {e}")
        return await handler(event, data)
```

Даже если что-то пошло не так в самой мидлвари, событие дойдёт до хендлера. Лучше пропустить лишнее сообщение, чем уронить обработку целиком.

## Когда использовать мидлвари

- **Проверка прав** (админ, подписка, бан).
- **Антифлуд** (не чаще одного сообщения в N секунд).
- **Логирование событий** (что пришло, от кого, когда).
- **Инъекция общих зависимостей** (база, конфиг, API-клиент).
- **Замеры времени** обработки.
- **Обработка исключений** централизованно (см. ниже).

## Глобальный обработчик ошибок

Отдельная тема, но упомянем здесь: помимо мидлварей, у `Dispatcher` есть `errors` — обработчик исключений. Это не мидлварь, но решает похожую задачу — централизованно перехватить ошибку и не дать боту упасть.

```python
import logging
from aiogram.types import ErrorEvent

@dp.errors()
async def global_error_handler(event: ErrorEvent):
    logging.exception(f"Unhandled error: {event.exception}")
    # Можно отправить сообщение пользователю
    if event.update.message:
        await event.update.message.answer("Произошла ошибка, попробуй позже")
    return True  # ошибка обработана, не логировать повторно
```

Это спасение от ситуации, когда необработанное исключение в одном хендлере ломает обработку всего Update'а.

## Чего не должно быть в мидлвари

Мидлварь не должна содержать бизнес-логику. Её задача — **подготовить данные** или **проверить условие**, а не решать, что ответить пользователю в конкретном случае.

Плохо:

```python
class BadMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        if event.text == "привет":
            await event.answer("Привет!")  # это задача хендлера
            return
        return await handler(event, data)
```

Хорошо:

```python
class GoodMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        data["user"] = await get_user(event.from_user.id)  # подготовка данных
        return await handler(event, data)
```

Правило простое: если ваша мидлварь знает что-то специфичное про конкретное сообщение — скорее всего, эта логика должна быть в хендлере.

## Совет

Не плодите мидлвари без необходимости. Каждая мидлварь — это дополнительный слой, который нужно держать в голове при отладке. Если задача решается фильтром или хендлером — решайте там.

Если у вас много мидлварей с одинаковой логикой (например, проверка роли), вынесите их в отдельный модуль и подключайте пачкой:

```python
from middlewares import ThrottlingMiddleware, LoggingMiddleware, DatabaseMiddleware

def setup_middlewares(dp: Dispatcher, db):
    dp.update.outer_middleware(ThrottlingMiddleware(rate_limit=0.5))
    dp.update.outer_middleware(LoggingMiddleware())
    dp.update.outer_middleware(DatabaseMiddleware(db))
```

Тогда в `main()` останется одна строка `setup_middlewares(dp, db)` — код станет чище.
