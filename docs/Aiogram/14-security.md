# 14. Безопасность

Бот — это программа, которая работает от вашего имени, имеет доступ к базе, принимает пользовательский ввод и часто управляет деньгами или правами. Ошибки в безопасности здесь стоят дорого. Разберём базовые правила, которые уберегут от типичных проблем.

## Токен бота

Самое важное правило: **токен — это пароль от вашего бота**. Кто его знает — тот может делать от имени бота что угодно: писать пользователям, менять меню, удалять сообщения.

### Как хранить токен

- **В переменных окружения.** Никогда в коде, `.env` в репозиторий не коммитится.
- **Разные токены для dev и prod.** Если утечёт dev-токен — прод не пострадает.
- **Не логируйте токен.** Если у вас есть логирование конфигов — исключите поле `token`.
- **Ограничьте доступ к `.env`.** На сервере — `chmod 600 .env`, только владелец может читать.

Правильный `main.py`:

```python
from config import settings  # читает из .env

bot = Bot(token=settings.bot_token)
```

Неправильный:

```python
bot = Bot(token="123456:AAHdq...")  # токен в коде
```

### Если токен утёк

Немедленно отзовите его через `@BotFather`:

```
/revoke
```

Выберите бота из списка. Старый токен перестанет работать, вы получите новый. Все, кто им пользовался (включая ваш прод), потеряют доступ — обновите `.env` и перезапустите бота.

Проверьте, что утекло и куда. Если токен попал в публичный GitHub — недостаточно его отозвать: бота уже могли добавить в спам-каналы. Проверьте `@BotFather` → `/mybots` → выберите бота → `Bot Settings` → посмотрите список чатов через `/getchat`.

## callback_data

`callback_data` — это данные, которые приходят от пользователя при нажатии Inline-кнопки. Как и любой пользовательский ввод, их **нельзя доверять**.

### Проблема

Пользователь может подделать `callback_data`. Пример уязвимости:

```python
@router.callback_query(F.data.startswith("delete:"))
async def delete_post(callback: CallbackQuery, db):
    post_id = int(callback.data.split(":")[1])
    await db.execute("DELETE FROM posts WHERE id = $1", post_id)
    await callback.answer("Удалено")
```

Здесь нет проверки, что пользователь имеет право удалять пост. Он может нажать кнопку на чужом сообщении (если она там есть) или подделать `callback_data` через своего бота — и удалить чужой контент.

### Решение

Всегда проверяйте права на сервере:

```python
@router.callback_query(F.data.startswith("delete:"))
async def delete_post(callback: CallbackQuery, db):
    try:
        post_id = int(callback.data.split(":")[1])
    except (IndexError, ValueError):
        await callback.answer("Некорректные данные", show_alert=True)
        return

    # Проверяем, что пост существует и принадлежит пользователю
    post = await db.fetchrow(
        "SELECT author_id FROM posts WHERE id = $1", post_id,
    )
    if post is None:
        await callback.answer("Пост не найден", show_alert=True)
        return
    if post["author_id"] != callback.from_user.id:
        await callback.answer("Это не твой пост", show_alert=True)
        return

    await db.execute("DELETE FROM posts WHERE id = $1", post_id)
    await callback.answer("Удалено")
```

Правило простое: **никогда не доверяйте `callback_data` как источнику прав**. Он говорит, *что* пользователь хочет сделать, но не *может ли* он это сделать.

### Не храните секреты в callback_data

`callback_data` видна пользователю, если он откроет JSON кнопки через сторонние клиенты. Никогда не кладите туда пароли, токены, email и другие чувствительные данные.

## SQL-инъекции

SQL-инъекция — классическая уязвимость, когда пользовательский ввод попадает в SQL-запрос напрямую.

### Плохо

```python
# f-строка с пользовательским вводом
name = message.text
await db.execute(f"SELECT * FROM users WHERE full_name = '{name}'")
```

Если пользователь введёт `'; DROP TABLE users; --`, запрос станет:

```sql
SELECT * FROM users WHERE full_name = ''; DROP TABLE users; --'
```

И таблица исчезнет.

### Хорошо

asyncpg и SQLAlchemy используют **параметризованные запросы**. Значения передаются отдельно, они никогда не попадают в текст SQL.

```python
# параметризованный запрос
name = message.text
await db.fetch("SELECT * FROM users WHERE full_name = $1", name)
```

Для SQLAlchemy:

```python
# ORM не даёт сделать инъекцию
from sqlalchemy import select

stmt = select(User).where(User.full_name == name)
result = await session.execute(stmt)
```

Никогда не собирайте SQL через f-строки или `.format()`. Всегда передавайте параметры.

## Проверка прав

Каждый хендлер, который делает что-то важное (бан, удаление, рассылка), должен проверять права. Лучше всего — на уровне фильтра или мидлвари.

### Проверка админа

```python
from aiogram.filters import BaseFilter
from aiogram.types import Message

class IsAdmin(BaseFilter):
    def __init__(self, admin_ids: list[int]):
        self.admin_ids = admin_ids

    async def __call__(self, message: Message) -> bool:
        return message.from_user.id in self.admin_ids
```

Использование:

```python
@router.message(IsAdmin(settings.admin_ids), Command("ban"))
async def cmd_ban(message: Message, command: CommandObject):
    user_id = int(command.args)
    await message.bot.ban_chat_member(message.chat.id, user_id)
    await message.answer("Забанен")
```

Права храните в `.env` (через `pydantic-settings`):

```env
ADMIN_IDS=123456789,987654321
```

```python
class Settings(BaseSettings):
    admin_ids: list[int] = []

    model_config = SettingsConfigDict(env_file=".env")
```

### Проверка владельца ресурса

Не только админы имеют права. Обычный пользователь имеет право только на свои ресурсы.

```python
@router.callback_query(F.data.startswith("edit_profile:"))
async def edit_profile(callback: CallbackQuery, db):
    user_id = int(callback.data.split(":")[1])

    # Пользователь может редактировать только свой профиль
    if user_id != callback.from_user.id:
        await callback.answer("Это не твой профиль", show_alert=True)
        return

    # ... редактирование ...
```

Проверка всегда на сервере. Даже если вы «уверены», что кнопку мог нажать только владелец — она могла остаться в чате от другого пользователя, или пользователь мог переслать сообщение с кнопкой.

## Валидация пользовательского ввода

Всё, что приходит от пользователя, — потенциально опасно. Валидируйте:

- **Тип данных.** Если ждёте число — преобразуйте через `int()` в `try/except`.
- **Длина.** Проверьте, что сообщение не длиннее разумного (например, 4000 символов — лимит Telegram).
- **Формат.** Email, телефон, дата — проверяйте регулярками.
- **Диапазон.** ID мероприятия — в пределах допустимых значений.

```python
@router.message(Command("register"))
async def cmd_register(message: Message, command: CommandObject):
    if not command.args:
        await message.answer("Укажи ID мероприятия")
        return

    try:
        event_id = int(command.args)
    except ValueError:
        await message.answer("ID должен быть числом")
        return

    if event_id <= 0:
        await message.answer("ID должен быть положительным")
        return

    # ... продолжаем ...
```

Это защищает не только от взлома, но и от случайных ошибок пользователя.

## Проверка подписки на канал

Частый сценарий: бот требует подписку на канал. Проверять нужно **на сервере**, а не доверять сообщению пользователя «я подписался».

```python
class SubscriptionMiddleware(BaseMiddleware):
    def __init__(self, channel_id: int):
        self.channel_id = channel_id

    async def __call__(self, handler, event: Message, data: dict):
        bot: Bot = data["bot"]
        member = await bot.get_chat_member(self.channel_id, event.from_user.id)
        if member.status in ("left", "kicked"):
            await event.answer("Подпишись на канал!")
            return
        return await handler(event, data)
```

`get_chat_member` — единственный надёжный способ проверить статус. Не кешируйте результат надолго — пользователь может отписаться через минуту.

## Webhook: secret_token

Если вы используете webhook, обязательно настройте `secret_token`. Без него любой может отправить поддельный Update на ваш эндпоинт.

### Установка

```bash
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://your.app/webhook&secret_token=<SECRET>"
```

`<SECRET>` — любая строка из символов `A-Z`, `a-z`, `0-9`, `_`, `-` длиной 1–256.

### Проверка в коде

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
SECRET = "your-secret"

@app.post("/webhook")
async def webhook(request: Request):
    if request.headers.get("X-Telegram-Bot-Api-Secret-Token") != SECRET:
        raise HTTPException(status_code=403)
    update = await request.json()
    await dp.feed_raw_update(bot, update)
    return {"ok": True}
```

Без этой проверки любой человек может `curl`-ом отправить фейковый Update и заставить бота делать что угодно от имени любого пользователя.

## Проверка Login Widget

Если вы используете Telegram Login Widget (кнопка «Войти через Telegram» на сайте), данные приходят с хешем. Хеш нужно проверить — иначе злоумышленник подделает данные.

Алгоритм:

1. Уберите поле `hash` из полученных данных.
2. Отсортируйте оставшиеся поля по имени.
3. Соберите строку `key=value` через `\n`.
4. Секретный ключ = `SHA256(bot_token)`.
5. Вычислите `HMAC-SHA256` от строки с этим ключом.
6. Сравните с полученным `hash`.

```python
import hashlib
import hmac

def check_telegram_auth(data: dict, bot_token: str) -> bool:
    received_hash = data.pop("hash", None)
    if not received_hash:
        return False

    data_check_string = "\n".join(
        f"{k}={v}" for k, v in sorted(data.items())
    )
    secret_key = hashlib.sha256(bot_token.encode()).digest()
    computed_hash = hmac.new(
        secret_key, data_check_string.encode(), hashlib.sha256,
    ).hexdigest()

    return hmac.compare_digest(computed_hash, received_hash)
```

`hmac.compare_digest` — сравнение с постоянным временем, защищает от timing-атак. Обычное `==` использовать нельзя.

## Anti-flood и rate limiting

Без ограничений пользователь может спамить бота сотнями сообщений в секунду. Это забивает event loop, съедает ресурсы и мешает остальным.

Минимальная защита — `ThrottlingMiddleware` из главы 8.

Более серьёзная — **rate limiting на уровне API**. Telegram имеет лимиты:

- **30 сообщений/сек** в целом по боту.
- **1 сообщение/сек** в один чат.
- **20 сообщений/мин** в одну группу.

Если бот превышает лимит — Telegram вернёт `RetryAfter` с указанием, сколько ждать. Нужно уметь это обрабатывать:

```python
from aiogram.exceptions import TelegramRetryAfter

async def send_safe(bot: Bot, chat_id: int, text: str):
    try:
        await bot.send_message(chat_id, text)
    except TelegramRetryAfter as e:
        await asyncio.sleep(e.retry_after)
        await bot.send_message(chat_id, text)
```

Для больших рассылок используйте очередь с ограничением скорости — например, `aiolimiter` или собственную реализацию.

## Секреты в логах

Следите, что попадает в логи. Токены, пароли, содержимое `.env` — не должны там оказаться.

Если используете `logging.basicConfig` с `%`-форматированием аргументов — всё ок. Если конкатенируете строки с секретами — плохо.

```python
# секрет в логе
logging.info(f"Connecting with token {bot_token}")

# без секрета
logging.info("Connecting to Telegram")
```

Если используете Sentry или другой сборщик ошибок — настройте `before_send`, чтобы вычищать чувствительные данные.

## Обработка исключений

Не показывайте пользователю внутренние ошибки. Стек вызовов, имена таблиц, SQL-запросы — всё это подсказки для атакующего.

```python
@dp.errors()
async def error_handler(event: ErrorEvent, bot: Bot):
    logging.exception(f"Unhandled error: {event.exception}")  # полный стек — в лог

    if event.update.message:
        await event.update.message.answer(
            "Что-то пошло не так. Попробуй позже."  # без стека
        )

    return True
```

В лог — максимум информации. Пользователю — минимум.

## Зависимости

Следите за обновлениями библиотек. Уязвимости регулярно находят в `aiohttp`, `pydantic`, `asyncpg`. Обновляйтесь хотя бы раз в месяц.

```bash
pip install pip-audit
pip-audit
```

`pip-audit` проверяет зависимости на известные уязвимости и подсказывает, что обновить.

## Чек-лист безопасности

Перед деплоем проверьте:

- Токен в `.env`, `.env` в `.gitignore`, `.env` не в git-истории.
- Разные токены для dev и prod.
- Все SQL-запросы параметризованы (`$1`, `$2`, не f-строки).
- `callback_data` валидируется перед использованием.
- Права проверяются на сервере, а не подразумеваются.
- Пользовательский ввод проходит валидацию.
- Webhook защищён `secret_token`.
- Login Widget проверяет хеш через `hmac.compare_digest`.
- Есть антифлуд мидлварь.
- Ошибки не показывают стек пользователю.
- Секреты не попадают в логи.
- `pip-audit` не находит критичных уязвимостей.

## Совет

Самая частая ошибка — доверие `callback_data`. Если вы проверяете права только по кнопке, которую «мог нажать только админ», — считайте, что прав нет вообще. Всегда проверяйте `callback.from_user.id` на сервере.

Вторая частая ошибка — SQL-инъекция через f-строку. Если видите `f"SELECT ... {variable}"` — это красный флаг. Всегда параметризуйте.
