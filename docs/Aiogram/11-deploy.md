# 11. Деплой

Локальный запуск — только половина дела. Бота нужно выложить куда-то, чтобы он работал круглосуточно и не зависел от вашего ноутбука.

## Варианты

**Amvera Cloud** — отечественный аналог Heroku, дружелюбный к новичкам. Не требует настройки Linux: достаточно подключить репозиторий и сделать `git push`. Дают 111 рублей на тестирование.

**Vercel** — бесплатный вариант для лёгких ботов. Работает только через webhook, лимит 10 секунд на функцию, нет постоянного диска. Подходит для ботов, которые не занимаются тяжёлыми задачами.

**VPS** — полный контроль за 300–600 рублей в месяц. Свобода, но нужно уметь работать с Linux.

## Amvera

Создайте файл `amvera.yml` в корне проекта:

```yaml
run:
  command: python aiogram_run.py
  persistenceMount: /data
  containerPort: 8080
  nodeSelector:
    cpu: 100m
    memory: 128Mi
```

Затем подключите GitHub-репозиторий в панели Amvera и запушьте изменения.

## Vercel

Для Vercel нужен webhook вместо polling. Создайте файл `api/webhook.py` с FastAPI:

```python
from fastapi import FastAPI, Request
from aiogram import Bot, Dispatcher

app = FastAPI()
bot = Bot(token="...")
dp = Dispatcher()

@app.post("/webhook")
async def webhook(request: Request):
    update = await request.json()
    await dp.feed_raw_update(bot, update)
    return {"ok": True}
```

После деплоя установите webhook одной командой:

```bash
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://your-project.vercel.app/webhook"
```

## VPS

На VPS обычно настраивают автозапуск через systemd. Создайте файл `/etc/systemd/system/mybot.service`:

```ini
[Unit]
Description=Telegram Bot
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/mybot
ExecStart=/home/botuser/mybot/.venv/bin/python aiogram_run.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Затем — активируйте сервис:

```bash
sudo systemctl enable mybot
sudo systemctl start mybot
```
