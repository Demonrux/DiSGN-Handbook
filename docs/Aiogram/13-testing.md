# 13. Тестирование ботов

Бот растёт, хендлеров становится всё больше, и однажды вы меняете фильтр — и ломаете три сценария, не заметив. Тесты ловят такие ситуации до деплоя. Разберёмся, как тестировать aiogram-ботов без боли.

## Почему это сложнее, чем кажется

Наивный подход — запустить бота и вручную потыкать кнопки. Работает, но:

- Медленно. Полный прогон вручную — десятки минут.
- Ненадёжно. Легко забыть проверить сценарий, который трогали месяц назад.
- Не автоматизируется. Нельзя встроить в CI/CD.

Правильный подход — писать автоматические тесты, которые подают боту **сырые Update'ы** и проверяют ответы. Никакого Telegram, никаких реальных сообщений.

## Два уровня тестирования

Тесты делятся на два уровня.

**Unit-тесты** проверяют отдельные функции в изоляции. Хендлер — это async-функция, её можно вызвать напрямую с мок-объектом сообщения. Быстро, но не проверяет роутинг и фильтры.

**Integration-тесты** проверяют всю цепочку: Update → Dispatcher → Router → Handler. Медленнее, но ловят проблемы, которые unit-тесты не видят.

Хорошая стратегия — писать оба. Unit — для сложной логики внутри хендлера. Integration — для ключевых сценариев (регистрация, запись на мероприятие, оплата).

## Подготовка окружения

Установите зависимости для тестов:

```bash
pip install pytest pytest-asyncio
```

Создайте `pytest.ini` в корне проекта:

```ini
[pytest]
asyncio_mode = auto
testpaths = tests
```

`asyncio_mode = auto` — важная настройка. Без неё придётся на каждом тесте ставить `@pytest.mark.asyncio`. С ней pytest-asyncio сам определяет асинхронные тесты.

## Unit-тест хендлера

Простейший случай: у нас есть хендлер, который отвечает на сообщение. Проверим, что он вызвал `answer` с правильным текстом.

```python
# handlers.py
async def echo_handler(message):
    await message.answer(f"Ты написал: {message.text}")
```

```python
# tests/test_handlers.py
from unittest.mock import AsyncMock
import pytest

from handlers import echo_handler

async def test_echo_handler():
    message = AsyncMock()
    message.text = "привет"

    await echo_handler(message)

    message.answer.assert_awaited_once_with("Ты написал: привет")
```

Здесь `AsyncMock` — это заглушка, которая притворяется объектом `Message`. У неё есть все атрибуты (`text`, `from_user`, `chat`), и все методы возвращают awaitable-моки. Мы задаём нужные поля вручную и проверяем, что хендлер вызвал `answer` с правильным аргументом.

`assert_awaited_once_with` — проверка, что метод был вызван ровно один раз с указанными аргументами.

## Проверка сложной логики

Если хендлер содержит валидацию, проверьте все ветки.

```python
# handlers.py
async def process_email(message, state):
    email = message.text.strip()
    if "@" not in email or "." not in email:
        await message.answer("Это не похоже на email")
        return
    await state.update_data(email=email)
    await message.answer("Спасибо!")
```

```python
# tests/test_handlers.py
import pytest
from unittest.mock import AsyncMock

async def test_email_valid():
    message = AsyncMock()
    message.text = "  ivan@example.com  "
    state = AsyncMock()

    await process_email(message, state)

    state.update_data.assert_awaited_once_with(email="ivan@example.com")
    message.answer.assert_awaited_once_with("Спасибо!")

async def test_email_invalid():
    message = AsyncMock()
    message.text = "не-имейл"
    state = AsyncMock()

    await process_email(message, state)

    state.update_data.assert_not_called()
    message.answer.assert_awaited_once_with("Это не похоже на email")
```

Тесты покрывают обе ветки: корректный email и некорректный. Если однажды кто-то случайно уберёт проверку — тест поймает.

## Integration-тест через feed_raw_update

Unit-тесты хороши, но не проверяют роутинг. Что если вы случайно подключили роутер в неправильном порядке и хендлер не срабатывает? Нужен тест на весь пайплайн.

`Dispatcher.feed_raw_update()` позволяет подать сырой JSON-Update напрямую в диспетчер — как будто он пришёл от Telegram.

```python
import time
import pytest
from aiogram import Bot, Dispatcher, F
from aiogram.types import Message

@pytest.fixture
def bot():
    return Bot(token="42:TEST")

@pytest.fixture
def dp():
    return Dispatcher()

async def test_dispatcher_routes_message(bot, dp):
    @dp.message(F.text == "ping")
    async def ping_handler(message: Message):
        return "pong"

    result = await dp.feed_raw_update(
        bot=bot,
        update={
            "update_id": 1,
            "message": {
                "message_id": 1,
                "date": int(time.time()),
                "text": "ping",
                "chat": {"id": 42, "type": "private"},
                "from": {
                    "id": 42,
                    "is_bot": False,
                    "first_name": "Test",
                },
            },
        },
    )

    assert result == "pong"
```

Что здесь важно:

- **`feed_raw_update`** возвращает то, что вернул хендлер. Можно проверять результат напрямую.
- **JSON собирается руками.** Да, громоздко, но зато тест не зависит ни от каких сетевых вызовов.
- **`date: int(time.time())`** — Telegram требует свежий timestamp, а `aiogram` может отбрасывать старые Update'ы. Свежее время спасает.

## Фикстура для Update

Собирать JSON руками каждый раз утомительно. Вынесем в фикстуру.

```python
# tests/conftest.py
import time
import pytest

@pytest.fixture
def make_update():
    def _make(text: str, user_id: int = 42, chat_type: str = "private"):
        return {
            "update_id": 1,
            "message": {
                "message_id": 1,
                "date": int(time.time()),
                "text": text,
                "chat": {"id": user_id, "type": chat_type},
                "from": {
                    "id": user_id,
                    "is_bot": False,
                    "first_name": "Test",
                    "username": "test_user",
                },
            },
        }
    return _make
```

Использование:

```python
async def test_help_command(bot, dp, make_update):
    @dp.message(F.text == "/help")
    async def help_handler(message):
        return "help shown"

    result = await dp.feed_raw_update(bot=bot, update=make_update("/help"))
    assert result == "help shown"
```

Тесты стали короче и читаемее.

## Проверка callback_query

Callback'и тоже легко тестировать.

```python
async def test_callback_route(bot, dp):
    @dp.callback_query(F.data == "confirm")
    async def confirm_handler(callback):
        return "confirmed"

    result = await dp.feed_raw_update(
        bot=bot,
        update={
            "update_id": 2,
            "callback_query": {
                "id": "abc",
                "from": {"id": 42, "is_bot": False, "first_name": "Test"},
                "chat_instance": "xxx",
                "data": "confirm",
                "message": {
                    "message_id": 10,
                    "date": int(time.time()),
                    "chat": {"id": 42, "type": "private"},
                    "text": "Нажми",
                },
            },
        },
    )

    assert result == "confirmed"
```

## Мок-зависимости

В реальном боте хендлеры работают с БД, API и другими внешними сервисами. Тесты не должны ходить в настоящую БД — это медленно и хрупко.

Мок-мидлварь или явная передача мока в `data` решает проблему.

```python
class FakeDB:
    def __init__(self):
        self.users = {}

    async def execute(self, query, *args):
        pass

    async def fetchrow(self, query, *args):
        return self.users.get(args[0])

    async def fetch(self, query, *args):
        return list(self.users.values())
```

Тест:

```python
async def test_user_registered(bot, dp, make_update):
    fake_db = FakeDB()
    fake_db.users[42] = {"telegram_id": 42, "full_name": "Иван"}

    @dp.message(F.text == "/me")
    async def me_handler(message, db):
        user = await db.fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            message.from_user.id,
        )
        return user["full_name"] if user else "не зарегистрирован"

    result = await dp.feed_raw_update(
        bot=bot,
        update=make_update("/me"),
        db=fake_db,  # передаём как зависимость
    )

    assert result == "Иван"
```

`feed_raw_update` принимает дополнительные kwargs — они попадут в `data` и будут доступны в хендлере. Это тот же механизм, что и с мидлварями.

## Тестирование FSM

FSM требует `FSMContext`. Его тоже можно создать вручную.

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage

async def test_registration_flow():
    storage = MemoryStorage()
    state = FSMContext(storage=storage, key=StorageKey(bot_id=1, chat_id=1, user_id=1))

    # ... вызываете хендлеры, передавая state ...
    # и проверяете, что состояние и данные меняются как надо
```

Но проще через `feed_raw_update` — если FSM-хранилище подключено к диспетчеру, `state` создаётся автоматически. Проверить состояние после обработки можно напрямую через хранилище:

```python
async def test_state_after_command(bot, dp, make_update):
    storage = MemoryStorage()
    dp = Dispatcher(storage=storage)

    @dp.message(F.text == "/start")
    async def start_handler(message, state):
        await state.set_state("waiting_name")

    await dp.feed_raw_update(bot=bot, update=make_update("/start"))

    key = StorageKey(bot_id=bot.id, chat_id=42, user_id=42)
    state_data = await storage.get_state(key=key)
    assert state_data.state == "waiting_name"
```

## Что стоит тестировать, а что нет

Не всё нужно покрывать тестами. Разделяйте по стоимости и ценности.

**Тестировать обязательно:**

- Логику с ветвлениями (валидация, права, состояния).
- Обработку ошибок (что происходит, если БД вернула `None`).
- Ключевые бизнес-сценарии (регистрация, оплата, бронирование).
- Граничные случаи (пустой текст, неверный формат данных).

**Не тестировать:**

- Простую передачу данных (хендлер «отправил ответ — тест не нужен»).
- Библиотеки и фреймворк — их тестируют авторы.
- Команды, где логика — один вызов `bot.send_message`.

Правило: тестируйте то, что может сломаться незаметно. Если изменение в коде не может сломать поведение — тест не нужен.

## Запуск и покрытие

Запуск всех тестов:

```bash
pytest
```

Запуск одного файла:

```bash
pytest tests/test_handlers.py
```

Запуск одного теста:

```bash
pytest tests/test_handlers.py::test_email_valid
```

Покрытие кода:

```bash
pip install pytest-cov
pytest --cov=. --cov-report=term-missing
```

`term-missing` покажет не только процент, но и конкретные строки, которые не покрыты тестами. Это полезнее, чем просто число.

Целевой процент покрытия — тема холиварная. Начинайте с 40–50% для ключевой логики, не гонитесь за 100% любой ценой.

## CI/CD

Тесты особенно полезны в CI. Пример `.github/workflows/test.yml`:

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pip install pytest pytest-asyncio pytest-cov
      - run: pytest --cov=. --cov-report=term-missing
```

Каждый пуш в репозиторий автоматически прогоняет тесты. Если что-то сломалось — узнаете до того, как код попадёт в прод.

## Совет

Не стремитесь тестировать всё. Начните с одного теста — на самый важный сценарий. Если он работает, добавьте второй. Постепенно покроете критичные места.

Хорошая отправная точка: при каждом баге заводите тест, который его воспроизводит, а потом фиксите баг. Так вы гарантированно не поймаете ту же проблему дважды.
