# 3. Архитектура aiogram: Bot, Dispatcher, Router

Вся работа aiogram строится на трёх объектах. Понимание их ролей — ключ к пониманию фреймворка.

## Bot — «трубка связи» с Telegram

`Bot` — это объект, который умеет отправлять запросы в Telegram API. Он знает токен, умеет делать HTTP-запросы и парсить ответы. Через него вы отправляете сообщения, редактируете их, удаляете, баните пользователей, ставите реакции.

`Bot` ничего не знает о логике вашего бота. Он не решает, что делать при новом сообщении — он просто умеет разговаривать с Telegram.

```python
from aiogram import Bot
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

bot = Bot(
    token="...",
    default=DefaultBotProperties(parse_mode=ParseMode.HTML)
)
```

Параметр `default` задаёт настройки по умолчанию — например, режим разметки текста. Это удобно: не нужно каждый раз передавать `parse_mode` в `send_message`.

## Dispatcher — «мозг» бота

`Dispatcher` принимает поток Update'ов от Telegram и решает, какой хендлер должен обработать каждый из них. Он хранит список роутеров, знает про мидлвари, FSM-хранилище и глобальные обработчики ошибок.

Если `Bot` — это руки, то `Dispatcher` — это мозг. Он смотрит на Update и спрашивает у каждого зарегистрированного хендлера: «Ты хочешь это обработать?». Первый, кто ответил «да», получает событие.

```python
from aiogram import Dispatcher
from aiogram.fsm.storage.memory import MemoryStorage

dp = Dispatcher(storage=MemoryStorage())
```

`Dispatcher` — единственный на весь бот. Создаётся один раз в `main()` и живёт до остановки.

## Router — «отдел» внутри мозга

`Router` — это контейнер для хендлеров. Он позволяет разбить логику бота на независимые модули: один роутер отвечает за регистрацию, другой — за админку, третий — за уведомления. Каждый роутер можно разрабатывать и тестировать отдельно.

```python
from aiogram import Router
from aiogram.filters import CommandStart
from aiogram.types import Message

start_router = Router()

@start_router.message(CommandStart())
async def cmd_start(message: Message):
    await message.answer("Привет!")
```

Затем роутер подключается к диспетчеру:

```python
dp.include_router(start_router)
```

С этого момента все хендлеры роутера становятся частью диспетчера.

## Иерархия роутеров

Роутеры можно вкладывать друг в друга, создавая иерархию. Это позволяет сначала проверять общие условия, а потом — частные.

```
Dispatcher
├── Router: admin_router
│   ├── handler: /admin
│   ├── handler: /ban
│   └── Router: admin_events_router
│       └── handler: /admin/events
├── Router: start_router
│   ├── handler: /start
│   └── handler: /help
└── Router: fallback_router
    └── handler: (все остальные сообщения)
```

Вложенность полезна, когда у вас много хендлеров и нужно сгруппировать их по смыслу. Например, все команды администратора сначала проходят через `admin_router`, где стоит фильтр на `admin_ids`, а уже потом попадают в конкретный хендлер.

```python
from aiogram import Router, F
from aiogram.filters import Command

# Создаём родительский роутер
admin_router = Router()
# Фильтр на уровне роутера: сюда попадут только админы
admin_router.message.filter(F.from_user.id.in_([123, 456]))

# Создаём дочерний роутер
admin_events_router = Router()

@admin_events_router.message(Command("events"))
async def cmd_events(message: Message):
    await message.answer("Список мероприятий (админ)")

# Вкладываем дочерний в родительский
admin_router.include_router(admin_events_router)

# Подключаем к диспетчеру
dp.include_router(admin_router)
```

**Важно:** роутеры нельзя зацикливать. Если роутер A включает B, а B включает A — aiogram выбросит ошибку при подключении.

## Порядок подключения роутеров

Порядок, в котором вы вызываете `dp.include_router(...)`, определяет порядок проверки. Это критично для маршрутизации.

```python
# Правильно: сначала специфичные, потом общие
dp.include_router(admin_router)
dp.include_router(start_router)
dp.include_router(fallback_router)

# Неправильно: fallback перехватит всё
dp.include_router(fallback_router)
dp.include_router(admin_router)  # никогда не вызовется
```

Правило простое: **чем специфичнее роутер, тем раньше он должен быть подключён**.

## Как всё работает вместе

Когда пользователь пишет боту, происходит такая цепочка:

1. Telegram получает сообщение и формирует Update.
2. Bot забирает Update (через polling или webhook).
3. Dispatcher передаёт Update в первый роутер.
4. Роутер проверяет свои хендлеры по порядку — каждый хендлер имеет фильтры.
5. Хендлер, чьи фильтры совпали, вызывается.
6. Внутри хендлера вы через `message.answer()` или `bot.send_message()` отвечаете пользователю.

Если ни один хендлер не подошёл — Update просто игнорируется. Ни ошибки, ни предупреждения в консоли не будет, если вы не настроили логирование.

## Полный пример main.py

Вот как выглядит типичная точка входа в бота:

```python
import asyncio
import logging

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage

from routers import admin_router, start_router, fallback_router

async def main():
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )

    bot = Bot(
        token="...",
        default=DefaultBotProperties(parse_mode=ParseMode.HTML),
    )
    dp = Dispatcher(storage=MemoryStorage())

    # Порядок подключения важен
    dp.include_router(admin_router)
    dp.include_router(start_router)
    dp.include_router(fallback_router)

    # Удаляем накопившиеся Update'ы при старте (важно при разработке)
    await bot.delete_webhook(drop_pending_updates=True)

    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
```

Разберём ключевые моменты:

- `logging.basicConfig` — включает логи. Без него вы не увидите ошибок в хендлерах.
- `DefaultBotProperties(parse_mode=ParseMode.HTML)` — задаёт HTML-разметку по умолчанию.
- `MemoryStorage` — хранилище FSM. Для разработки сойдёт, для продакшена нужен Redis.
- `delete_webhook(drop_pending_updates=True)` — сбрасывает накопившиеся Update'ы. Полезно при рестарте: старые сообщения не будут обработаны повторно.
- `dp.start_polling(bot)` — запускает бесконечный цикл получения Update'ов.

## Почему это удобно

Такое разделение — не просто красивая абстракция. Оно даёт несколько важных преимуществ.

**Bot можно легко заменить.** Например, использовать локальный сервер Bot API для обхода лимитов — достаточно передать другой `base_url`.

**Dispatcher можно переиспользовать.** Например, в тестах: вы создаёте `Dispatcher`, подключаете роутер и подаёте ему сырые Update'ы через `feed_raw_update()`. Никакого реального Telegram не нужно.

**Роутеры делают код модульным.** Большой бот из двадцати хендлеров перестаёт быть кашей и превращается в набор независимых блоков. Каждый роутер можно вынести в отдельный файл:

```
project/
├── aiogram_run.py
├── config.py
├── routers/
│   ├── __init__.py
│   ├── start.py
│   ├── events.py
│   └── admin.py
├── filters/
├── middlewares/
└── database.py
```

Импортируете роутеры из модулей и подключаете в `main()`.

## Совет

Порядок подключения роутеров легко забыть. Если хендлер «не срабатывает» — первым делом проверьте, не перехватывает ли его роутер, подключённый раньше. Включите `logging` на уровне `INFO` для `aiogram.dispatcher` — увидите, какие Update'ы приходят и каким хендлерам передаются.
