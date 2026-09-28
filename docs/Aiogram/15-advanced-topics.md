# 15. Продвинутые темы

Мы разобрали всё, что нужно для 90% ботов. Эта глава — обзор того, что находится за пределами базового курса. Здесь нет глубоких примеров: цель — показать, куда двигаться дальше и какие инструменты для этого есть.

## Inline-режим

Inline-режим позволяет вызывать бота через `@username` в **любом** чате — не только в личке с ботом. Пользователь пишет `@mybot погода`, бот предлагает варианты, пользователь выбирает один — и сообщение уходит в чат.

Типичные применения: отправка стикеров, гифок, мемов, шаблонов сообщений, переводов, ссылок.

Точка входа — событие `inline_query`:

```python
from aiogram.types import InlineQuery, InlineQueryResultArticle, InputTextMessageContent

@router.inline_query()
async def inline_handler(query: InlineQuery):
    results = [
        InlineQueryResultArticle(
            id="1",
            title="Привет",
            input_message_content=InputTextMessageContent(
                message_text="Привет!"
            ),
        ),
    ]
    await query.answer(results, cache_time=5)
```

Ключевые моменты:

- **`cache_time`** — сколько секунд Telegram может кешировать результаты. Меньше значение — свежее данные, но выше нагрузка.
- **`InlineQueryResultArticle`** — простой текстовый результат. Есть ещё `InlineQueryResultPhoto`, `InlineQueryResultGif` и другие.
- **`id`** — уникальный идентификатор результата. Повторяющиеся `id` приведут к ошибке.

Inline-режим нужно включить у `@BotFather`: `/setinline`.

## Telegram Mini Apps (Web Apps)

Mini Apps — это веб-приложения, встроенные внутрь Telegram. Пользователь открывает их как отдельное окно поверх чата. Внутри — обычный веб-интерфейс, который взаимодействует с ботом через JS-мост.

Зачем это нужно:

- **Сложные интерфейсы.** Формы, каталоги, карты, дашборды — всё, что не влезает в формат чата.
- **Платёжный флоу.** Оплата проходит красиво, без кнопок в чате.
- **Интеграция с сайтом.** Можно переиспользовать существующий веб-фронтенд.

Бот открывает WebApp кнопкой:

```python
from aiogram.types import WebAppInfo, InlineKeyboardMarkup, InlineKeyboardButton

kb = InlineKeyboardMarkup(inline_keyboard=[[
    InlineKeyboardButton(
        text="Открыть приложение",
        web_app=WebAppInfo(url="https://app.example.com"),
    ),
]])

await message.answer("Открой мини-приложение:", reply_markup=kb)
```

Внутри приложения — Telegram WebApp SDK: `window.Telegram.WebApp`. Он даёт доступ к данным пользователя, тему, кнопки «назад», «закрыть» и отправку данных обратно боту.

Данные от WebApp приходят как `message.web_app_data` — их нужно валидировать так же, как и Login Widget (через HMAC).

## Платежи Telegram Stars

> **Опционально.** Этот раздел — для ботов, которые что-то продают
> (подписки, доступ к контенту, товары). Если ваш бот ничего не продаёт —
> смело пропускайте.

Telegram Stars — встроенная валюта Telegram для оплаты в ботах. Работает в App Store, Google Play, Web — без привязки к платёжным системам.

Флоу оплаты:

1. **Бот отправляет счёт** через `send_invoice`.
2. **Пользователь оплачивает** — Telegram показывает встроенное окно.
3. **Бот получает `pre_checkout_query`** — нужно подтвердить, что всё готово к оплате.
4. **Бот получает `successful_payment`** — в `message` с типом `successful_payment`.
5. **Бот выдаёт товар.**

```python
from aiogram.types import LabeledPrice, PreCheckoutQuery

@router.message(Command("buy"))
async def cmd_buy(message: Message):
    await message.answer_invoice(
        title="Премиум доступ",
        description="Месяц без рекламы",
        payload="premium_1month",
        currency="XTR",  # XTR = Telegram Stars
        prices=[LabeledPrice(label="Премиум", amount=100)],
    )

@router.pre_checkout_query()
async def pre_checkout(query: PreCheckoutQuery):
    # Здесь можно проверить, что товар ещё в наличии
    await query.answer(ok=True)

@router.message(F.successful_payment)
async def on_payment(message: Message, db):
    payment = message.successful_payment
    await db.execute(
        "INSERT INTO payments (user_id, payload, amount) VALUES ($1, $2, $3)",
        message.from_user.id, payment.invoice_payload, payment.total_amount,
    )
    await message.answer("Оплата прошла! Спасибо.")
```

**Важно:** `pre_checkout_query` нужно подтвердить в течение 10 секунд, иначе платёж отменится. `successful_payment` может прийти с задержкой или вообще не прийти — на этот случай нужны ретраи и проверка через `getStarTransactions`.

## Локализация (i18n)

Если бот для нескольких стран — понадобится локализация. Есть библиотеки: `aiogram-i18n`, `fluentogram`. Или собственное решение через словари.

Минимальная реализация:

```python
TEXTS = {
    "ru": {"hello": "Привет, {name}!"},
    "en": {"hello": "Hello, {name}!"},
}

def t(lang: str, key: str, **kwargs) -> str:
    return TEXTS.get(lang, TEXTS["ru"])[key].format(**kwargs)
```

Язык хранится в БД у пользователя или в FSM. В хендлере:

```python
@router.message(CommandStart())
async def cmd_start(message: Message, db):
    user = await db.fetchrow(
        "SELECT language FROM users WHERE telegram_id = $1",
        message.from_user.id,
    )
    lang = user["language"] if user else message.from_user.language_code or "ru"
    await message.answer(t(lang, "hello", name=message.from_user.first_name))
```

Для сложных проектов `aiogram-i18n` даёт больше: поддержку `.ftl`-файлов, автоматическое определение языка, плюрализацию.

## Работа с лимитами Telegram

Telegram строго следит за нагрузкой. Основные лимиты:

| Ограничение | Лимит |
|---|---|
| Сообщений в секунду в один чат | 1 |
| Сообщений в минуту в группу | 20 |
| Общая скорость сообщений бота | 30/сек |
| Длина сообщения | 4096 символов |
| Размер `callback_data` | 64 байта |
| Размер файла для скачивания | 20 МБ |
| Размер файла для отправки | 50 МБ |

При превышении лимита Telegram возвращает ошибку `RetryAfter` с указанием, сколько секунд подождать.

Для массовых рассылок нужна **очередь** с контролем скорости. Простая реализация через `asyncio.Queue` и один воркер:

```python
async def sender_worker(bot: Bot):
    while True:
        chat_id, text = await queue.get()
        try:
            await bot.send_message(chat_id, text)
        except TelegramRetryAfter as e:
            await asyncio.sleep(e.retry_after)
            await queue.put((chat_id, text))  # вернуть в очередь
        except TelegramForbiddenError:
            pass  # пользователь заблокировал бота
        finally:
            queue.task_done()
        await asyncio.sleep(1 / 30)  # не более 30 сообщений в секунду
```

Запускается как фоновая задача через `asyncio.create_task`. Бот остаётся отзывчивым, а рассылка идёт с нужной скоростью.

## Локальный Bot API Server

> **Опционально.** Локальный сервер нужен только при работе с большими
> файлами, высокой нагрузке или специфических требованиях к хранению
> данных. Для 95% ботов хватит официального API.

Официальный Bot API ограничен: файлы до 20 МБ на скачивание, 50 МБ на отправку, нет возможности читать чужие сообщения в группах. Локальный сервер снимает эти лимиты.

Работает так: вы запускаете `telegram-bot-api` рядом с ботом, и все запросы идут через него, а не через `api.telegram.org`.

Запуск через Docker:

```bash
docker run -d --name telegram-bot-api \
  -p 8081:8081 \
  -v /path/to/data:/var/lib/telegram-bot-api \
  -e TELEGRAM_API_ID=... \
  -e TELEGRAM_API_HASH=... \
  aiogram/telegram-bot-api:latest \
  --local
```

В боте:

```python
from aiogram.client.session.aiohttp import AiohttpSession

session = AiohttpSession(api="http://localhost:8081")
bot = Bot(token=..., session=session)
```

`API_ID` и `API_HASH` получаются на my.telegram.org. Локальный сервер может работать и через webhook — тогда Telegram шлёт Update'ы прямо на ваш сервер.

Когда нужен локальный сервер:

- Файлы больше стандартных лимитов.
- Высокая нагрузка (снижает latency).
- Требования по хранению данных (не отдавать Telegram).
- Продвинутые сценарии (чтение истории, работа с секретными чатами).

## Микросервисная архитектура

Когда бот вырастает, его разбивают на несколько сервисов:

- **Bot-сервис** — принимает Update'ы, отвечает пользователям.
- **Worker-сервис** — выполняет тяжёлые задачи (рассылки, обработка, ML).
- **API-сервис** — отдаёт данные мобильному приложению, вебу.

Общаются через **брокер сообщений** (Redis, RabbitMQ, Kafka) или HTTP.

Плюсы:

- **Масштабирование.** Bot-сервисов может быть несколько, worker-сервисов — сколько нужно.
- **Изоляция.** Падение одного сервиса не роняет остальные.
- **Разные языки.** Можно писать часть на Python, часть на Go.

Минусы:

- **Сложность.** Больше инфраструктуры, больше мест, где что-то может сломаться.
- **Отладка.** Проблема может быть в любом звене цепочки.

Для небольших ботов это оверинжиниринг. Для ботов с сотнями тысяч пользователей — необходимость.

## Observability

Когда бот работает у сотен пользователей, хочется знать: сколько запросов, сколько ошибок, где узкие места. Для этого есть три столпа observability.

**Логи.** Структурированные (`structlog`), собираются в одном месте (`Loki`, `ELK`).

**Метрики.** `Prometheus` + `Grafana`. Пример метрик: количество сообщений в секунду, latency хендлеров, ошибки по типам.

**Трейсинг.** `OpenTelemetry` + `Jaeger`. Показывает путь одного запроса через все сервисы.

Минимальный пример с Prometheus:

```python
from prometheus_client import Counter, Histogram

messages_total = Counter("bot_messages_total", "Всего сообщений", ["handler"])
handler_duration = Histogram("bot_handler_seconds", "Время хендлера", ["handler"])

class MetricsMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        name = handler.__name__
        messages_total.labels(handler=name).inc()
        with handler_duration.labels(handler=name).time():
            return await handler(event, data)
```

Метрики отдаются по HTTP на `/metrics`, Prometheus их собирает, Grafana рисует графики. Ошибки видно в реальном времени, а не после жалоб пользователей.

## Feature flags

Когда функций становится много, полезно уметь включать и выключать их без деплоя. Feature flags — простой способ это сделать.

```python
class Features:
    def __init__(self, db):
        self.db = db

    async def is_enabled(self, name: str) -> bool:
        row = await self.db.fetchrow(
            "SELECT enabled FROM features WHERE name = $1", name,
        )
        return bool(row and row["enabled"])
```

Использование:

```python
@router.message(Command("beta"))
async def cmd_beta(message: Message, features):
    if not await features.is_enabled("beta_program"):
        await message.answer("Бета-программа пока недоступна")
        return
    # ...
```

Флаги хранятся в БД или Redis, меняются из админки, действуют мгновенно. Можно включать функцию только для части пользователей — например, для тестирования на 10%.

## Что дальше

Этот курс покрывает фундамент. Дальше есть несколько направлений:

- **Углубиться в aiogram.** Хорошо изучить `aiogram` по документации: диаграммы, глубины, кастомизации.
- **Изучить Python-экосистему.** `pydantic`, `sqlalchemy`, `asyncio`, `httpx` — пригодятся везде.
- **Изучить инфраструктуру.** Docker, Linux, systemd, PostgreSQL, Redis — база для продакшена.
- **Изучить конкретную область.** Если бот для магазина — изучить платежи, если для сообщества — модерацию, если для образования — LMS.
- **Изучить архитектуру.** DDD, Clean Architecture, CQRS — для больших ботов.

Главное — не останавливаться. Каждый новый бот будет требовать новых знаний, и это нормально.

## Совет

Не гонитесь за всеми темами сразу. Сделайте одного простого бота от начала до конца — с БД, деплоем, базовой безопасностью. Потом второго, посложнее. Опыт приходит только через практику, а не через чтение документации.

И второе: подглядывайте в open-source. На GitHub есть сотни открытых Telegram-ботов на aiogram. Читать чужой код — это тоже обучение.
