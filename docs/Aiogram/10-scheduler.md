# 10. Планировщик задач

Иногда бот должен действовать сам: присылать напоминания, чистить старые данные, рассылать анонсы. Для этого используется планировщик задач.

## APScheduler

**APScheduler** — библиотека, которая выполняет функции по расписанию внутри asyncio-цикла aiogram. Она не требует отдельного процесса: задачи живут в том же event loop, что и бот.

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler(timezone="Europe/Moscow")
```

## Триггеры

Задача запускается по одному из трёх триггеров.

**Interval** — каждые N секунд/минут/часов.

```python
scheduler.add_job(my_func, "interval", seconds=30)
```

**Cron** — в определённое время, как системный cron.

```python
scheduler.add_job(my_func, "cron", hour=10, minute=0)
```

**Date** — один раз в конкретный момент.

```python
scheduler.add_job(my_func, "date", run_date="2026-10-01 18:00")
```

## Пример: напоминание

```python
async def send_reminder(bot: Bot):
    await bot.send_message(chat_id=123456789, text="⏰ Напоминание!")

async def main():
    scheduler.add_job(send_reminder, "cron", hour=10, minute=0, args=(bot,))
    scheduler.start()
    await dp.start_polling(bot)
```

## Важные правила

**Не запускайте длительные операции.** Если задача займёт больше пары секунд, она заблокирует event loop, и бот перестанет отвечать. Для тяжёлых задач используйте `asyncio.to_thread()` или выносите их в отдельный воркер.

**Не забывайте про идемпотентность.** Задача может выполниться дважды — например, если бот запущен в нескольких экземплярах. Продумывайте логику так, чтобы повторное выполнение не приводило к дублям.

**Логируйте ошибки.** Если задача упадёт, вы узнаете об этом только из логов. Оборачивайте вызовы в try/except и записывайте всё в logger.
