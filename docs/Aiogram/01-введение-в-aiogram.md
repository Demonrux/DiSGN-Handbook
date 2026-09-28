# Telegram-боты на aiogram: от идеи до продакшена

Практическое руководство для тех, кто хочет разобраться в современном aiogram и завести своего первого бота за один вечер. Материал рассчитан на людей, знакомых с Python, но не обязательно с асинхронностью или Telegram API.

---

## Оглавление

- [Введение](#введение)
- [Создание бота через BotFather](#создание-бота-через-botfather)
- [Структура проекта](#структура-проекта)
- [Настройка окружения](#настройка-окружения)
- [Инициализация бота и диспетчера](#инициализация-бота-и-диспетчера)
- [Роутеры и фильтры](#роутеры-и-фильтры)
- [Магические фильтры](#магические-фильтры)
- [Клавиатуры](#клавиатуры)
- [Работа с базами данных](#работа-с-базами-данных)
- [Машина состояний FSM](#машина-состояний-fsm)
- [Планировщик задач](#планировщик-задач)
- [Мидлвари](#мидлвари)
- [Кастомные фильтры](#кастомные-фильтры)
- [Деплой](#деплой)
- [Лучшие практики](#лучшие-практики)

---

## Введение

`aiogram` — это асинхронный фреймворк для создания Telegram-ботов на Python. Он построен вокруг `asyncio`, поддерживает роутеры, магические фильтры, FSM, мидлвари и всё, что нужно для бота любого размера — от простой болванки до многоуровневой системы с базой данных, планировщиком и админкой.

Это руководство — попытка собрать в одном месте всё, что нужно знать для старта: от создания бота в BotFather до деплоя в облако и работы с базой данных. Мы будем двигаться последовательно, разбирая каждый шаг не только с точки зрения «как», но и «почему именно так».

---

## Создание бота через BotFather

Любой Telegram-бот начинается с регистрации у **@BotFather** — официального бота, который управляет всеми остальными. Открывайте его в Telegram, нажимайте «Старт» и отправляйте команду `/newbot`.

Дальше вас проведут через два шага. Сначала нужно придумать **отображаемое имя** — то, что пользователи увидят в шапке чата. Оно может быть на русском, содержать эмодзи и пробелы, и его всегда можно поменять. Затем — **username**, то есть адрес бота. Здесь правила жёстче: логин должен быть уникальным на весь Telegram и обязательно заканчиваться на `bot` в любом регистре. Изменить его потом уже нельзя, так что выбирайте вдумчиво.

Когда бот создан, BotFather пришлёт вам **токен** — длинную строку вида `720000:AAG2000Aa70p6eotkKiMKJD_nwosSAfvg`. Это ключ ко всему вашему боту: кто угодно с этим токеном может управлять ботом от вашего имени. Храните его в секрете, не коммитьте в публичные репозитории, а если он всё-таки утёк — немедленно отзывайте через команду `/revoke`.

---

## Структура проекта

Хорошая структура экономит часы отладки и месяцы рефакторинга. Вот проверенный временем шаблон, который прошёл через десятки проектов и легко расширяется по мере роста.

```text
my_project/
├── db_handler/
│   ├── __init__.py
│   └── db_class.py
├── handlers/
│   ├── __init__.py
│   ├── start.py
│   └── admin.py
├── keyboards/
│   ├── __init__.py
│   └── all_keyboards.py
├── work_time/
│   ├── __init__.py
│   └── time_func.py
├── utils/
│   ├── __init__.py
│   └── my_utils.py
├── filters/
│   ├── __init__.py
│   └── is_admin.py
├── middlewares/
│   ├── __init__.py
│   └── check_sub.py
├── .env
├── aiogram_run.py
├── create_bot.py
├── requirements.txt
└── run.py
```

Каждый пакет отвечает за свою зону ответственности. В `handlers` живут обработчики — здесь обычно разбивают логику на файлы по темам: старт, админка, пользовательские сценарии. `keyboards` хранит все клавиатуры, и в небольших проектах их удобно держать в одном файле. Пакет `db_handler` содержит универсальный класс для работы с PostgreSQL, а в `work_time` лежат функции, которые запускаются планировщиком по расписанию. Утилиты, фильтры и мидлвари вынесены в отдельные директории, потому что рано или поздно их становится много и они перестают умещаться в одном файле.

Такая структура кажется избыточной для бота из трёх команд, но она окупается уже на второй неделе разработки, когда хочется добавить админку, FSM-сценарий и пару фоновых задач.

---

## Настройка окружения

Секреты не должны попадать в код. Токен бота, ID администраторов, строка подключения к базе — всё это хранится в файле `.env`, который никогда не коммитится в репозиторий.

```env
TOKEN=720000:AAG2000Aa70p6eotkKiMKJD_nwosSAfvg
ADMINS=123456789,987654321
PG_LINK=postgresql://user:password@host:5432/dbname
```

Читаются эти переменные через `python-decouple` — библиотеку, которая умеет подгружать `.env` автоматически и предоставляет удобную функцию `config()`. Она возвращает строки, а если нужно число или список — преобразование делается вручную.

Зависимости описываются в `requirements.txt`. Минимальный набор для современного бота выглядит так:

```txt
aiogram>=3.28.0
asyncpg
APScheduler
python-decouple
```

`aiogram` — сам фреймворк, `asyncpg` — асинхронный драйвер PostgreSQL, `APScheduler` — планировщик задач, `python-decouple` — работа с переменными окружения. Установка стандартная:

```bash
pip install -r requirements.txt
```

---

## Инициализация бота и диспетчера

Файл `create_bot.py` — точка входа в мир aiogram. Здесь создаются три ключевых объекта: `Bot`, `Dispatcher` и `AsyncIOScheduler`. Они импортируются во всех остальных модулях, что позволяет не плодить дубликаты.

```python
import logging
from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from decouple import config
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# Инициализация БД (если нужна)
# from db_handler.db_class import PostgresHandler
# pg_db = PostgresHandler(config('PG_LINK'))

scheduler = AsyncIOScheduler(timezone='Europe/Moscow')
admins = [int(admin_id) for admin_id in config('ADMINS').split(',')]

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

bot = Bot(
    token=config('TOKEN'),
    default=DefaultBotProperties(parse_mode=ParseMode.HTML)
)
dp = Dispatcher(storage=MemoryStorage())
```

Обратите внимание на `DefaultBotProperties`. Настройки бота задаются отдельным объектом — так гибче, потому что можно управлять разными параметрами независимо. `ParseMode.HTML` означает, что текст сообщений будет интерпретироваться как HTML, поэтому можно использовать `<b>жирный</b>`, `<i>курсив</i>`, `<code>моноширинный</code>` и ссылки `<a href="...">текст</a>`.

`Dispatcher` — это центральный объект, который принимает все обновления от Telegram и решает, какому хендлеру их передать. Параметр `storage=MemoryStorage()` говорит, что состояния FSM будут храниться в оперативной памяти. Это работает, но не переживает перезапуск бота — для продакшена используется `RedisStorage`.

---

## Роутеры и фильтры

Роутеры — это способ организовать хендлеры в модули. Каждый файл создаёт собственный `Router()` и наполняет его обработчиками, а затем роутер подключается к диспетчеру один раз в точке входа. Такой подход избавляет от необходимости импортировать глобальный `dp` в каждый файл и делает код действительно модульным.

```python
from aiogram import Router, F
from aiogram.filters import CommandStart, Command
from aiogram.types import Message

start_router = Router()

@start_router.message(CommandStart())
async def cmd_start(message: Message):
    await message.answer('Привет! Я бот на aiogram')

@start_router.message(Command('help'))
async def cmd_help(message: Message):
    await message.answer('Доступные команды: /start, /help')

@start_router.message(F.text == 'привет')
async def cmd_hello(message: Message):
    await message.answer('Привет-привет!')
```

Подключение роутера происходит в `aiogram_run.py`:

```python
import asyncio
from create_bot import bot, dp, scheduler
from handlers.start import start_router

async def main():
    # scheduler.start()  # если используешь планировщик
    dp.include_router(start_router)
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
```

Функция `bot.delete_webhook(drop_pending_updates=True)` сбрасывает очередь накопившихся обновлений при старте. Это полезно во время разработки: без неё бот начнёт отвечать на сообщения, присланные пока он был выключен.

Роутеры можно вкладывать друг в друга для создания иерархии: например, `admin_router` внутри `start_router` — но зацикливать их нельзя, aiogram это проверит и упадёт с ошибкой.

---

## Магические фильтры

Магические фильтры `F` — один из самых приятных инструментов aiogram. Они позволяют описывать условия прямо в декораторе, без лямбд и вспомогательных функций.

```python
from aiogram import F

@router.message(F.text == 'меню')
@router.message(F.text.startswith('заказ'))
@router.message(F.text.endswith('.jpg'))
@router.message(F.text.isdigit())
@router.message(F.photo)
@router.message(F.video)
@router.message(F.document)
@router.message(F.from_user.id == 123456789)
```

Работает это через перегрузку операторов: `F.text` возвращает объект, который сравнивается с правой частью выражения и превращается в предикат. Условия можно комбинировать через `&` (логическое И) и `|` (логическое ИЛИ):

```python
@router.message(F.text.startswith('show') & F.text.endswith('example'))
@router.message(F.text == 'hi' | F.text == 'hello')
```

Магические фильтры читаются как естественный язык и режут типовой код почти вдвое. Это одна из тех мелочей, из-за которых в aiogram приятно возвращаться.

---

## Клавиатуры

В Telegram есть два принципиально разных типа клавиатур, и новички их часто путают. **Reply-клавиатура** заменяет системную клавиатуру внизу экрана, и нажатие на её кнопку отправляет текст как обычное сообщение. **Inline-клавиатура** прикрепляется к конкретному сообщению, и нажатие на её кнопку отправляет боту колбэк с указанными данными, не появляясь в чате.

Reply удобны для навигации — например, постоянно висящее меню с кнопками «Профиль», «Мероприятия», «Помощь». Inline подходят для контекстных действий: «Записаться на это мероприятие», «Отменить регистрацию», «Показать подробнее».

```python
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder

def get_reply_kb():
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text='📅 Сегодня')],
            [KeyboardButton(text='📋 Мои мероприятия'), KeyboardButton(text='👤 Профиль')]
        ],
        resize_keyboard=True
    )

def get_inline_kb():
    builder = InlineKeyboardBuilder()
    builder.button(text='📋 Мои регистрации', callback_data='my_regs')
    builder.button(text='📅 Мероприятия', callback_data='events')
    builder.button(text='👤 Профиль', callback_data='profile')
    builder.adjust(2)
    return builder.as_markup()
```

`InlineKeyboardBuilder` — удобный конструктор: можно добавлять кнопки одну за другой, а метод `adjust()` расставит их по рядам. Параметр `resize_keyboard=True` в reply-клавиатуре заставляет её занимать минимально возможное место, что визуально приятнее.

---

## Работа с базами данных

Для ботов, которые хранят что-то больше приветственного сообщения, нужна база данных. `PostgreSQL` — стандартный выбор: надёжно, бесплатно, огромное сообщество. Связка с aiogram идёт через **asyncpg** — асинхронный драйвер, который не блокирует event loop.

```python
import asyncpg

class PostgresHandler:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(self.dsn)

    async def execute(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.execute(query, *args)

    async def fetch(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetch(query, *args)

    async def fetchrow(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.fetchrow(query, *args)
```

Ключевая идея — **пул соединений**. Открывать новый коннект на каждый запрос дорого, поэтому `create_pool` создаёт набор переиспользуемых соединений, а `async with self.pool.acquire()` берёт одно из них на время операции и возвращает обратно.

Классическая схема для бота-регистратора на мероприятия выглядит так:

```sql
CREATE TABLE users (
    telegram_id BIGINT PRIMARY KEY,
    full_name   TEXT NOT NULL,
    group_name  TEXT,
    email       TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE events (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    description TEXT,
    event_date  TIMESTAMPTZ NOT NULL,
    location    TEXT,
    registration_open BOOLEAN DEFAULT TRUE
);

CREATE TABLE registrations (
    user_id  BIGINT REFERENCES users(telegram_id),
    event_id INT REFERENCES events(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, event_id)
);
```

Три таблицы закрывают 90% задач: пользователи, мероприятия и связь между ними «многие ко многим». Такой дизайн легко расширяется — можно добавить поля для QR-кодов, листа ожидания или отзывов, не ломая существующую логику.

---

## Машина состояний FSM

FSM — **Finite State Machine**, или конечный автомат — это способ вести пошаговый диалог с пользователем. Классический пример: анкета, где бот спрашивает имя, потом возраст, потом город, и на каждом шаге ждёт определённого ответа.

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup

class Form(StatesGroup):
    name = State()
    age = State()

@router.message(Command('fill_form'))
async def start_form(message: Message, state: FSMContext):
    await message.answer('Как тебя зовут?')
    await state.set_state(Form.name)

@router.message(Form.name)
async def process_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer('Сколько тебе лет?')
    await state.set_state(Form.age)

@router.message(Form.age)
async def process_age(message: Message, state: FSMContext):
    data = await state.get_data()
    await message.answer(f'Приятно познакомиться, {data["name"]}!')
    await state.clear()
```

Состояние можно представлять как «где сейчас находится пользователь в сценарии». Фильтр `Form.name` срабатывает только если пользователь в этом состоянии, так что случайное «привет» в чате не собьёт бота с толку. Данные, собранные в процессе, сохраняются через `state.update_data()` и читаются через `state.get_data()`.

Одна из самых частых ошибок новичков — использовать `MemoryStorage` в продакшене. При перезапуске бота все состояния теряются, и пользователи, которые были в середине анкеты, оказываются в подвешенном состоянии. Для продакшена нужен `RedisStorage`:

```python
from aiogram.fsm.storage.redis import RedisStorage
storage = RedisStorage.from_url('redis://localhost:6379/0')
dp = Dispatcher(storage=storage)
```

---

## Планировщик задач

Боты редко ограничиваются реакцией на сообщения. Иногда им нужно действовать самостоятельно: присылать напоминания, чистить старые записи, рассылать анонсы. Для этого используется **APScheduler**, который работает прямо внутри asyncio-цикла aiogram.

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from aiogram import Bot

scheduler = AsyncIOScheduler(timezone='Europe/Moscow')

async def send_reminder(bot: Bot):
    await bot.send_message(chat_id=123456789, text='⏰ Напоминание!')

async def main():
    scheduler.add_job(send_reminder, 'cron', hour=10, minute=0, args=(bot,))
    scheduler.start()
    await dp.start_polling(bot)
```

Есть три типа триггеров. `interval` запускает задачу через равные промежутки времени — удобно для регулярной очистки или опроса внешних API. `cron` срабатывает в определённый момент — как системный cron в Linux. И `date` выполняет задачу один раз в указанное время, что полезно для отложенных напоминаний.

Одна тонкость: **не запускайте в планировщике длительные операции**. Если задача займёт больше пары секунд, она заблокирует event loop, и бот перестанет отвечать пользователям. Для тяжёлых задач используйте `asyncio.to_thread()` или выносите их в отдельный воркер.

---

## Мидлвари

Мидлварь — это промежуточный слой, который выполняется до обработчика. Представьте себе конвейер: сообщение от Telegram сначала попадает в диспетчер, затем в мидлвари, и только потом в конкретный хендлер. На каждом шаге можно что-то сделать: залогировать, проверить права, добавить данные в контекст.

Типичные сценарии для мидлварей — проверка подписки на канал, антифлуд, логирование запросов, инъекция зависимостей. Например, если у вас десять хендлеров используют объект базы данных, не нужно передавать его в каждый — мидлварь положит его в `data`, и он автоматически попадёт в аргументы хендлера.

```python
from aiogram import BaseMiddleware
from aiogram.types import Message

class CheckSubMiddleware(BaseMiddleware):
    async def __call__(self, handler, event: Message, data: dict):
        user_id = event.from_user.id
        # ... логика проверки подписки
        return await handler(event, data)

dp.message.middleware(CheckSubMiddleware())
```

Мидлвари можно подключать не только к диспетчеру, но и к отдельным роутерам — это позволяет применять проверку подписки только к пользовательской части, а админам оставить свободный доступ.

---

## Кастомные фильтры

Иногда встроенных фильтров не хватает. Например, нужно ограничить доступ к команде только администраторами. Для этого пишется собственный фильтр — класс, наследующий `BaseFilter`.

```python
from aiogram.filters import BaseFilter
from aiogram.types import Message
from create_bot import admins

class IsAdmin(BaseFilter):
    async def __call__(self, message: Message) -> bool:
        return message.from_user.id in admins

@router.message(IsAdmin(), Command('admin'))
async def admin_panel(message: Message):
    await message.answer('Добро пожаловать в админ-панель!')
```

Такой подход держит логику проверки в одном месте: если завтра понадобится добавить ещё одно условие — скажем, проверку, что пользователь не забанен — достаточно поправить класс, а не искать `if` по всему проекту.

---

## Деплой

Локальный запуск — только половина дела. Рано или поздно бота нужно куда-то выложить, чтобы он работал круглосуточно и не зависел от вашего ноутбука. Есть три разумных варианта, и выбор зависит от бюджета и задач.

**Amvera Cloud** — отечественный аналог Heroku, самый дружелюбный к новичкам. Не нужно настраивать Linux, SSH и systemd: достаточно создать файл `amvera.yml` в корне проекта, подключить GitHub-репозиторий и сделать `git push`. Сервис сам соберёт контейнер, запустит его и будет следить за состоянием. При регистрации дают 111 рублей на тестирование, чего хватит на пару недель плотной работы. Главное преимущество для российских разработчиков — не нужно возиться с прокси до Telegram API.

```yaml
---
run:
  command: python aiogram_run.py
  persistenceMount: /data
  containerPort: 8080
  nodeSelector:
    cpu: 100m
    memory: 128Mi
```

**Vercel** — отличный выбор для лёгких ботов, которые не занимаются тяжёлыми задачами. Бесплатный тариф Hobby не требует карты, деплой через GitHub делается за минуту, а интеграция с базами вроде Neon или Supabase — почти автоматическая. Но есть три жёстких ограничения. Во-первых, бот работает только через **webhook**, не через polling — то есть Telegram сам дёргает ваш эндпоинт при новом сообщении. Во-вторых, функция выполняется максимум **10 секунд** на бесплатном тарифе, что отсекает скачивание видео, парсинг сайтов и массовые рассылки. В-третьих, нет постоянного диска — SQLite использовать нельзя, нужна внешняя база.

**VPS** — золотой стандарт, если нужна полная свобода. За 300–600 рублей в месяц вы получаете виртуальную машину с root-доступом, ставите Python, PostgreSQL, Redis, настраиваете systemd — и бот работает ровно так, как вы хотите. Минус один: нужно уметь работать в терминале Linux. Если планируете серьёзный проект с массовыми рассылками, фоновыми задачами и файлами, VPS окупится быстро.

Универсальное правило: **начинайте с Amvera**, если нужен работающий бот к утру. Переходите на **Vercel**, если бот лёгкий и хочется бесплатно. И берите **VPS**, если чувствуете, что упёрлись в потолок платформ.

---

## Лучшие практики

Завершая руководство, хочется собрать воедино принципы, которые отличают проекты, живущие годами, от тех, что умирают через месяц.

**Используйте роутеры.** Один большой файл с двадцатью хендлерами — это боль. Разбейте логику на модули по темам: старт, админка, регистрация, профиль. Если файл превысил 300 строк, пора его делить.

**Типизируйте аргументы.** Аннотации `Message`, `CallbackQuery`, `FSMContext` помогают IDE понимать ваш код и подсказывать методы. Без них автодополнение работает плохо, и вы будете лазить в документацию за каждой мелочью.

**Не блокируйте event loop.** Это правило номер один для асинхронного кода. `requests`, `time.sleep`, синхронные запросы к БД — всё это замораживает бота. Используйте `aiohttp`, `asyncio.sleep` и асинхронные драйверы. Если библиотека только синхронная, оборачивайте вызов в `asyncio.to_thread()`.

**RedisStorage для FSM в продакшене.** `MemoryStorage` теряет состояния при перезапуске. Это не критично, пока вы разрабатываете локально, но на живом боте приведёт к потоку жалоб от пользователей, застрявших в середине анкеты.

**Очередь для массовых рассылок.** Отправка 300 сообщений в одном хендлере упрётся в лимиты Telegram и таймауты платформы. Правильный подход — складывать задачи в таблицу `broadcast_queue` и разгребать её по 20–25 сообщений раз в минуту через cron.

**Логируйте через `logging`, а не `print`.** Модуль `logging` умеет писать в файлы, фильтровать по уровням и форматировать сообщения. `print` этого не умеет и остаётся в коде только до первого продакшена.

**Храните секреты в `.env`.** И добавьте `.env` в `.gitignore`. Истории о ботах, которые слили токен в публичный репозиторий, случаются с завидной регулярностью.

**Обрабатывайте ошибки глобально.** Один необработанный `Exception` внутри хендлера — и пользователь не получит ответа. Подключите `dp.errors.register()` и логируйте все исключения, чтобы не гоняться за ними по логам вручную.

**Тестируйте локально через polling.** Polling не требует публичного URL, туннелей и SSL. Разрабатывайте и отлаживайте на своём компьютере, а на webhook переходите только при деплое.

**Читайте документацию.** [docs.aiogram.dev](https://docs.aiogram.dev) — лучший источник правды. Примеры в интернете часто устарели, а официальная документация всегда актуальна.

---

