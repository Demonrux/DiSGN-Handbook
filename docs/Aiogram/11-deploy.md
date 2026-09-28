# 11. Деплой

Локальный запуск — только половина дела. Бота нужно выложить куда-то, чтобы он работал круглосуточно и не зависел от вашего ноутбука.

## Варианты хостинга

Есть три принципиально разных подхода к размещению бота.

| Платформа | Стоимость | Тип | Особенности |
|---|---|---|---|
| Amvera Cloud | От 0 ₽ (тест) | PaaS | Простой деплой через Git, поддержка РФ |
| Vercel | Бесплатно | Serverless | Только webhook, лимит 10 сек, нет диска |
| Railway | От $5/мес | PaaS | Простой деплой, webhook из коробки |
| VPS | 300–600 ₽/мес | IaaS | Полный контроль, нужен Linux |

### Amvera Cloud

Отечественный аналог Heroku, дружелюбный к новичкам. Не требует настройки Linux: достаточно подключить репозиторий и сделать `git push`. Дают 111 рублей на тестирование.

Подходит, если вы не хотите возиться с сервером, но нужен полноценный бот с polling и БД.

### Vercel

Бесплатный вариант для лёгких ботов. Работает только через webhook, лимит 10 секунд на функцию, нет постоянного диска. Подходит для ботов, которые не занимаются тяжёлыми задачами.

Не подойдёт, если нужно: polling, локальная БД, фоновые задачи, файловая система.

### VPS

Полный контроль за 300–600 рублей в месяц. Свобода, но нужно уметь работать с Linux.

Оптимальный вариант, если планируете долгоживущего бота с БД, планировщиком и фоновыми задачами.

## Amvera

Создайте файл `amvera.yml` в корне проекта:

```yaml
meta:
  environment: python
  toolchain:
    name: pip
    version: "3.12"

build:
  requirementsPath: requirements.txt
  scriptName: aiogram_run.py

run:
  persistenceMount: /data
  containerPort: 8080
  nodeSelector:
    cpu: 100m
    memory: 128Mi
```

Затем подключите GitHub-репозиторий в панели Amvera и запушьте изменения. Amvera сама соберёт контейнер, установит зависимости и запустит бота.

**Важно:** переменные окружения (токен) задаются в панели Amvera, а не в `.env` — файла `.env` там не будет.

## Vercel

Для Vercel нужен webhook вместо polling. Создайте файл `api/webhook.py` с FastAPI:

```python
import os
from fastapi import FastAPI, Request, HTTPException
from aiogram import Bot, Dispatcher

app = FastAPI()

# На Vercel переменные окружения задаются в дашборде проекта.
# Локально их подхватывает .env — но в проде .env нет.
BOT_TOKEN = os.environ["BOT_TOKEN"]
SECRET_TOKEN = os.environ["WEBHOOK_SECRET"]

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

@app.post("/webhook")
async def webhook(request: Request):
    if request.headers.get("X-Telegram-Bot-Api-Secret-Token") != SECRET_TOKEN:
        raise HTTPException(status_code=403)
    update = await request.json()
    await dp.feed_raw_update(bot, update)
    return {"ok": True}
```

После деплоя установите webhook одной командой:

```bash
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://your-project.vercel.app/webhook"
```

Проверить, что webhook установлен:

```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

В ответе должно быть `"url": "https://your-project.vercel.app/webhook"` и `"pending_update_count": 0`.

### Защита webhook секретным токеном

Webhook — публичный эндпоинт. Любой, кто знает URL, может слать туда запросы. Чтобы Telegram мог подтвердить, что запрос от него, используют **secret_token**.

При установке webhook передайте `secret_token`:

```bash
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=...&secret_token=your-secret"
```

Telegram будет добавлять заголовок `X-Telegram-Bot-Api-Secret-Token` к каждому запросу. Проверьте его в обработчике:

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
SECRET_TOKEN = "your-secret"

@app.post("/webhook")
async def webhook(request: Request):
    if request.headers.get("X-Telegram-Bot-Api-Secret-Token") != SECRET_TOKEN:
        raise HTTPException(status_code=403)
    update = await request.json()
    await dp.feed_raw_update(bot, update)
    return {"ok": True}
```

Без этой проверки кто угодно может отправить в ваш webhook поддельный Update — и бот отреагирует, будто это реальное событие.

## VPS

На VPS обычно настраивают автозапуск через systemd. Это стандартный менеджер сервисов в Linux: если бот упадёт — systemd поднимет его снова, если сервер перезагрузится — бот запустится автоматически.

### Подготовка

Установите Python, pip и создайте пользователя для бота (запускать от root — плохая практика):

```bash
sudo apt update
sudo apt install python3.12 python3.12-venv
sudo adduser --system --group botuser
```

Склонируйте проект в `/home/botuser/mybot`, создайте виртуальное окружение и установите зависимости:

```bash
cd /home/botuser/mybot
python3.12 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

### systemd-сервис

Создайте файл `/etc/systemd/system/mybot.service`:

```ini
[Unit]
Description=Telegram Bot
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/mybot
EnvironmentFile=/home/botuser/mybot/.env
ExecStart=/home/botuser/mybot/.venv/bin/python aiogram_run.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Разберём ключевые строки:

- `User=botuser` — запуск от имени непривилегированного пользователя. Если бота взломают, злоумышленник не получит root.
- `EnvironmentFile` — systemd прочитает `.env` и передаст переменные в процесс. Токен не нужно вписывать в сам сервис-файл.
- `Restart=always` — перезапуск при любом завершении. Если бот упадёт из-за необработанной ошибки, systemd его поднимет.
- `RestartSec=5` — пауза между попытками. Без неё при циклической ошибке systemd будет перезапускать бота сотни раз в секунду.

Активируйте и запустите:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mybot
sudo systemctl start mybot
```

Проверить статус:

```bash
sudo systemctl status mybot
```

Посмотреть логи:

```bash
sudo journalctl -u mybot -f
```

Флаг `-f` — «follow», как `tail -f`. Логи пишутся в journald, отдельные файлы заводить не нужно.

Перезапустить после обновления кода:

```bash
cd /home/botuser/mybot
git pull
.venv/bin/pip install -r requirements.txt
sudo systemctl restart mybot
```

## Docker

Docker упрощает деплой: один раз описали окружение — работает везде.

`Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "aiogram_run.py"]
```

`docker-compose.yml` (удобно для локальной разработки с БД и Redis):

```yaml
services:
  bot:
    build: .
    restart: always
    env_file: .env
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: bot
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: botdb
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: always

volumes:
  pgdata:
```

Запуск:

```bash
docker compose up -d
```

`-d` — detached, в фоне. Логи: `docker compose logs -f bot`.

## Хранение секретов

Токен, пароль от БД, API-ключи — всё это секреты. Правила простые:

1. **Никогда не коммитить в Git.** Даже в приватный. История коммитов хранится вечно.
2. **`.env` в `.gitignore`.** Файл `.env.example` с фиктивными значениями — можно и нужно.
3. **Переменные окружения на проде.** В systemd — `EnvironmentFile`, в Docker — `env_file`, в Amvera — панель управления.
4. **Разные токены для dev и prod.** Если один утечёт — второй останется цел.

Пример `.env.example`:

```env
BOT_TOKEN=123456:REPLACE_ME
DATABASE_URL=postgresql://user:pass@localhost/db
ADMIN_IDS=[123,456]
REDIS_URL=redis://localhost:6379/0
```

## Логирование

По умолчанию `logging.basicConfig(level=logging.INFO)` пишет в stderr. На systemd это попадёт в journald, на Docker — в `docker logs`, на Vercel — в дашборд.

Для продакшена полезно писать логи в файл с ротацией:

```python
import logging
from logging.handlers import RotatingFileHandler

handler = RotatingFileHandler(
    "bot.log",
    maxBytes=10 * 1024 * 1024,  # 10 МБ
    backupCount=5,
    encoding="utf-8",
)
handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
))

logging.basicConfig(level=logging.INFO, handlers=[handler])
```

`RotatingFileHandler` сам создаёт новый файл при достижении лимита и удаляет старые. Иначе `bot.log` вырастет до гигабайтов и забьёт диск.

**Два важных момента про логи в проде.**

Первый — добавьте файл логов в `.gitignore`. Иначе после `git pull` на сервере
`bot.log` попадёт в diff, а при неаккуратном коммите — в репозиторий:

```gitignore
bot.log
bot.log.*
*.log
```

Второй — **в Docker логи в файл теряются при рестарте контейнера**, если файл
не лежит на смонтированном volume. Либо примонтируйте директорию:

```yaml
services:
  bot:
    build: .
    volumes:
      - ./logs:/app/logs
```

И в `RotatingFileHandler` пишите в `/app/logs/bot.log`, а не в `bot.log`.
Либо не заморачивайтесь с файлами и пишите в stdout — Docker и systemd
сами соберут логи (`docker logs`, `journalctl`).

## Мониторинг

Минимальный набор, чтобы знать, что бот жив:

1. **Проверка процесса.** `systemctl status mybot` — сервис должен быть `active (running)`.
2. **Свежесть логов.** Если в `journalctl -u mybot` нет записей уже час — что-то не так.
3. **Уведомление об ошибках.** Отправляйте критические ошибки в отдельный чат через `bot.send_message`.

Пример уведомления админу об ошибке:

```python
@dp.errors()
async def global_error_handler(event: ErrorEvent, bot: Bot):
    logging.exception(f"Unhandled error: {event.exception}")
    try:
        await bot.send_message(
            chat_id=ADMIN_CHAT_ID,
            text=f"Ошибка в боте:\n<code>{event.exception}</code>",
        )
    except Exception:
        pass
    return True
```

Для серьёзных проектов — **Sentry**, **Grafana**, **Prometheus**. Для учебного бота хватит уведомлений в Telegram.

## Проверка после деплоя

После первого деплоя обязательно проверьте:

- `/start` отвечает.
- Токен подхватился из окружения (в логах нет ошибок авторизации).
- БД доступна (если используется).
- Redis доступен (если используется).
- Планировщик запущен (в логах есть записи о регистрации задач).
- Webhook установлен, если используется (через `getWebhookInfo`).

Самый частый провал — забыть переменную окружения. Бот стартует, падает на `NoneType` при обращении к токену и тихо умирает. Проверьте логи первым делом.

## Совет

Начните с простого: Amvera или VPS с systemd. Не пытайтесь сразу настраивать Kubernetes, CI/CD и blue-green deployment. Для 99% ботов хватит systemd + git pull.

Если хочется автоматизации — GitHub Actions на пуш в `main` соберёт Docker-образ и запустит `systemctl restart mybot` через SSH. Но это уже следующий уровень.
