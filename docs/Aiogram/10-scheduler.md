# 10. Планировщик задач

Иногда бот должен действовать сам: присылать напоминания, чистить старые данные, рассылать анонсы. Для этого используется планировщик задач.

## APScheduler

**APScheduler** — библиотека, которая выполняет функции по расписанию внутри asyncio-цикла aiogram. Она не требует отдельного процесса: задачи живут в том же event loop, что и бот.

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler(timezone="Europe/Moscow")
```

Почему именно `AsyncIOScheduler`, а не `BackgroundScheduler`? Потому что `BackgroundScheduler` создаёт свои потоки, и вызов асинхронных функций из него — боль. `AsyncIOScheduler` работает в том же loop'е, что и aiogram, и может вызывать async-функции напрямую.

## Триггеры

Задача запускается по одному из трёх триггеров.

### Interval — каждые N секунд/минут/часов

```python
scheduler.add_job(my_func, "interval", seconds=30)
scheduler.add_job(my_func, "interval", minutes=5)
scheduler.add_job(my_func, "interval", hours=1)
```

Подходит для регулярных задач: проверка статуса, синхронизация, отправка сводок.

### Cron — в определённое время

```python
# Каждый день в 10:00
scheduler.add_job(my_func, "cron", hour=10, minute=0)

# По будням в 9:30
scheduler.add_job(my_func, "cron", day_of_week="mon-fri", hour=9, minute=30)

# Первое число каждого месяца в полночь
scheduler.add_job(my_func, "cron", day=1, hour=0, minute=0)
```

Cron-триггер понимает выражения вроде `day_of_week="mon-fri"`, `hour="*/2"` (каждые 2 часа), `month="1,7"` (январь и июль). Полный синтаксис — в документации APScheduler.

### Date — один раз в конкретный момент

```python
scheduler.add_job(my_func, "date", run_date="2026-10-01 18:00")
```

Или относительная дата:

```python
from datetime import datetime, timedelta

run_at = datetime.now() + timedelta(hours=2)
scheduler.add_job(my_func, "date", run_date=run_at)
```

Используется для одноразовых напоминаний: «через час после регистрации отправь письмо».

## Пример: напоминание

Простой пример — ежедневное напоминание в 10:00.

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from aiogram import Bot

async def send_reminder(bot: Bot, chat_id: int):
    await bot.send_message(chat_id=chat_id, text="Напоминание!")

async def main():
    # ... создание bot, dp ...

    scheduler = AsyncIOScheduler(timezone="Europe/Moscow")
    scheduler.add_job(
        send_reminder, "cron",
        hour=10, minute=0,
        args=(bot, 123456789),  # передаём аргументы
        id="daily_reminder",     # уникальный ID
        replace_existing=True,   # перезапуск без дублей
    )
    scheduler.start()

    await dp.start_polling(bot)
```

Ключевые моменты:

- `args=(bot, 123456789)` — аргументы, которые будут переданы функции.
- `id="daily_reminder"` — уникальный идентификатор задачи. Если задать его повторно с `replace_existing=True`, старая задача заменится новой.
- `scheduler.start()` — запуск. Обязательно **до** `dp.start_polling()`.

## Пример: напоминание всем пользователям

Задача посложнее — разослать напоминание всем зарегистрированным пользователям.

```python
import logging
from aiogram import Bot

async def send_daily_reminder(bot: Bot, db):
    users = await db.fetch("SELECT telegram_id FROM users WHERE reminders_enabled = TRUE")
    for user in users:
        try:
            await bot.send_message(
                user["telegram_id"],
                "Не забудь про мероприятие сегодня!",
            )
        except Exception as e:
            logging.error(f"Ошибка отправки {user['telegram_id']}: {e}")
```

Обратите внимание на `try/except` внутри цикла: если один пользователь заблокировал бота, отправка упадёт с `TelegramForbiddenError`. Без `try/except` цикл прервётся, и остальные не получат сообщение.

Подключение:

```python
scheduler.add_job(
    send_daily_reminder, "cron",
    hour=9, minute=0,
    args=(bot, db),
    id="daily_reminder",
    replace_existing=True,
)
```

## Пример: одноразовое напоминание через N минут

Иногда нужно напомнить пользователю через час после его действия. Например, если он записался на мероприятие — прислать напоминание за день до события.

```python
from datetime import datetime, timedelta

@router.callback_query(F.data.startswith("register:"))
async def on_register(callback: CallbackQuery, scheduler, bot):
    event_id = int(callback.data.split(":")[1])
    # ... записываем в БД ...

    event_date = datetime.fromisoformat("2026-10-15 18:00")
    remind_at = event_date - timedelta(days=1)

    scheduler.add_job(
        send_reminder,
        "date",
        run_date=remind_at,
        args=(bot, callback.from_user.id, event_id),
        id=f"remind_{callback.from_user.id}_{event_id}",
        replace_existing=True,
    )

    await callback.answer("Записал! Напомню за день до мероприятия.")
```

Через `id` мы избегаем дублей: если пользователь записался, отменил и записался снова, напоминание будет одно.

## Запуск и остановка

Планировщик нужно запустить в `main()` и (опционально) корректно остановить.

```python
async def main():
    # ... создание bot, dp, db ...

    scheduler = AsyncIOScheduler(timezone="Europe/Moscow")
    scheduler.start()

    # Передаём планировщик в хендлеры через мидлварь или data
    dp.update.outer_middleware(...)

    try:
        await dp.start_polling(bot)
    finally:
        scheduler.shutdown(wait=False)  # не ждать завершения текущих задач
```

Если `wait=True` — `shutdown()` заблокируется, пока не завершатся все запущенные задачи. Это может затянуть остановку бота. `wait=False` безопаснее: текущие задачи будут оборваны, но при следующем запуске состояние восстановится (если задачи идемпотентны — см. ниже).

## Передача зависимостей в задачи

Функции задач обычно нужны `bot`, `db` и другие объекты. Вариантов два.

**Через `args`:**

```python
scheduler.add_job(send_reminder, "cron", hour=10, minute=0, args=(bot, db))
```

Просто, но громоздко при большом количестве зависимостей.

**Через замыкание:**

```python
def make_reminder_task(bot, db):
    async def task():
        users = await db.fetch("...")
        for user in users:
            await bot.send_message(user["telegram_id"], "Напоминание")
    return task

scheduler.add_job(make_reminder_task(bot, db), "cron", hour=10, minute=0)
```

Замыкание захватывает зависимости и передаёт их в задачу. Такой подход удобнее, когда задач много и они используют общие объекты.

## Важные правила

**Не запускайте длительные операции.** Если задача займёт больше пары секунд, она заблокирует event loop, и бот перестанет отвечать. Для тяжёлых задач используйте `asyncio.to_thread()` или выносите их в отдельный воркер.

Плохо:

```python
async def heavy_task():
    result = compute_something_heavy()  # блокирует loop
    await bot.send_message(...)
```

Хорошо:

```python
async def heavy_task():
    result = await asyncio.to_thread(compute_something_heavy)  # в отдельном потоке
    await bot.send_message(...)
```

**Не забывайте про идемпотентность.** Задача может выполниться дважды — например, если бот запущен в нескольких экземплярах или произошёл сбой и рестарт. Продумывайте логику так, чтобы повторное выполнение не приводило к дублям.

Плохо:

```python
async def send_reminder():
    await bot.send_message(user_id, "Напоминание")  # может отправиться дважды
```

Хорошо:

```python
async def send_reminder():
    # Проверяем, не отправляли ли уже сегодня
    sent = await db.fetchval(
        "SELECT EXISTS(SELECT 1 FROM sent_reminders WHERE user_id = $1 AND date = CURRENT_DATE)",
        user_id,
    )
    if sent:
        return
    await bot.send_message(user_id, "Напоминание")
    await db.execute("INSERT INTO sent_reminders (user_id) VALUES ($1)", user_id)
```

Это особенно актуально, если бот развёрнут в нескольких инстансах — например, за балансировщиком. Обе копии запустят задачу одновременно.

**Логируйте ошибки.** Если задача упадёт, вы узнаете об этом только из логов. Оборачивайте вызовы в `try/except` и записывайте всё в logger.

```python
async def safe_task():
    try:
        await do_something()
    except Exception as e:
        logging.exception(f"Task failed: {e}")
```

## Мониторинг задач

APScheduler позволяет посмотреть, какие задачи запланированы:

```python
for job in scheduler.get_jobs():
    print(job.id, job.next_run_time)
```

Полезно для отладки: убедиться, что задача зарегистрировалась и знает, когда запустится.

## Когда APScheduler не подходит

APScheduler хорош для небольших ботов. Когда задач становится много, а надёжность критична, переходят на **отдельный воркер**:

- **Celery** + Redis/RabbitMQ — стандарт индустрии.
- **ARQ** — лёгкая альтернатива на asyncio.
- **Dramatiq** — простой и надёжный.

Бот в этом случае только кладёт задачи в очередь, а воркер их выполняет. Это даёт:

- **Изоляцию.** Тяжёлая задача не блокирует бота.
- **Масштабирование.** Можно поднять несколько воркеров.
- **Надёжность.** Задача не потеряется при падении.
- **Ретраи.** Автоматические повторные попытки при ошибке.

Для учебных проектов это избыточно. Но если бот будет рассылать тысячи сообщений — стоит задуматься заранее.

##  Совет

Одна задача — одна функция. Не пихайте в один job и рассылку, и очистку, и логирование. Если что-то упадёт, вы не поймёте, что именно. Разбейте на отдельные задачи с понятными `id`:

```python
scheduler.add_job(send_daily_reminder, "cron", hour=9, id="daily_reminder")
scheduler.add_job(cleanup_old_data, "cron", hour=3, id="cleanup")
scheduler.add_job(sync_external_api, "interval", minutes=15, id="sync")
```

Если в логах видите `Job "cleanup" raised an exception` — сразу ясно, куда смотреть.
