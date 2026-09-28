<div align="center">
  <img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/0aadcaff-e2d8-4b83-b7d5-5f2d0af6eed7" />
</div>

# Руководство по Telegram-ботам на aiogram

Полный курс по созданию Telegram-ботов на Python с использованием aiogram: от первого бота до продакшена.

## О курсе

Курс рассчитан на тех, кто уже знаком с Python, но никогда не писал ботов. Мы идём от базы к продвинутым темам: разбираем, как устроен Telegram Bot API, как работает aiogram, и доходим до деплоя, тестирования и безопасности.

В каждом разделе — теория, примеры кода и практические советы.

## Содержание

### Основы

1. [Что такое Telegram-бот и как он работает](01-what-is-a-bot.md)
   — Bot API, Update, Polling vs Webhook, BotFather и токен.
2. [Асинхронность и asyncio](02-asyncio-basics.md)
   — Event loop, `async`/`await`, что блокирует loop, `asyncio.gather`.
3. [Архитектура aiogram: Bot, Dispatcher, Router](03-architecture.md)
   — Три ключевых объекта, иерархия роутеров, точка входа.

### Логика бота

4. [Хендлеры и фильтры](04-handlers-and-filters.md)
   — Виды событий, встроенные и кастомные фильтры, порядок проверки.
5. [Магические фильтры F](/05-magic-filters.md)
   — Условия в декораторе, комбинации, частые ошибки.
6. [Клавиатуры](06-keyboards.md)
   — Reply vs Inline, `callback_data`, валидация.
7. [Машина состояний FSM](07-fsm.md)
   — Пошаговые диалоги, `StatesGroup`, Redis как хранилище.
8. [Мидлвари](08-middlewares.md)
   — Outer vs inner, антифлуд, инъекция зависимостей, проверка подписки.

### Данные и автоматизация

9. [Работа с базой данных](09-database.md)
   — PostgreSQL, asyncpg, пул соединений, SQLAlchemy, миграции.
10. [Планировщик задач](10-scheduler.md)
    — APScheduler, триггеры, идемпотентность, мониторинг.
11. [Деплой](11-deploy.md)
    — Amvera, Vercel, VPS, Docker, systemd, секреты.

### Дополнительно

12. [Форматирование текста и отправка медиа](12-formatting-and-media.md)
    — HTML-разметка, `FSInputFile`, альбомы, скачивание файлов.
13. [Тестирование ботов](13-testing.md)
    — Unit-тесты, `feed_raw_update`, моки, CI/CD.
14. [Безопасность](14-security.md)
    — Токен, SQL-инъекции, `callback_data`, secret_token, чек-лист.
15. [Продвинутые темы](15-advanced.md)
    — Inline-режим, Mini Apps, платежи, локализация, микросервисы.

