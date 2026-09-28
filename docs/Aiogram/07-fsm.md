# 7. Машина состояний FSM

FSM (Finite State Machine, конечный автомат) — это способ вести с пользователем пошаговый диалог, где на каждом шаге бот ждёт определённого ответа.

## Зачем нужна FSM

Представим, что бот запрашивает у пользователя анкету: имя, группу, email. Без FSM бот не понимает, что именно сейчас от него хотят. Пользователь отправит сообщение «Иван», и бот не знает — это имя, это группа или это случайный текст.

FSM решает эту проблему: бот помнит, на каком шаге находится каждый пользователь. Пока пользователь в состоянии «ожидаю имя» — любое сообщение считается именем. Пока в состоянии «ожидаю группу» — считается группой.

Схематично:

```
Пользователь A: /register → [waiting_name] → "Иван" → [waiting_group] → "ИУ7-42" → [waiting_email] → ...
Пользователь B: /register → [waiting_name] → "Мария" → [waiting_group] → ...
```

Состояние у каждого чата своё. Пользователи не мешают друг другу.

## Состояния

Состояния описываются через класс-наследник `StatesGroup`. Каждое поле — это отдельное состояние.

```python
from aiogram.fsm.state import State, StatesGroup

class Registration(StatesGroup):
    waiting_name = State()
    waiting_group = State()
    waiting_email = State()
```

Обычно класс называется по смыслу сценария, а состояния — по тому, чего бот ждёт от пользователя. Это делает код самодокументируемым: глядя на `Registration.waiting_email`, сразу понятно, что происходит.

Состояния могут быть сгруппированы в **группы** — так проще организовывать логику:

```python
class Registration(StatesGroup):
    waiting_name = State()
    waiting_group = State()

class EventBooking(StatesGroup):
    choosing_event = State()
    confirming = State()
```

Это разные сценарии — регистрация и запись на мероприятие. Они не пересекаются, и путаницы не возникает.

## Переключение состояний

У каждого чата есть текущее состояние. Переключить его можно через `state.set_state()`.

```python
from aiogram.fsm.context import FSMContext

@router.message(Command("register"))
async def start_registration(message: Message, state: FSMContext):
    await message.answer("Как тебя зовут?")
    await state.set_state(Registration.waiting_name)
```

Параметр `state: FSMContext` нужно добавить в аргументы хендлера — aiogram передаст его автоматически.

**Важно:** имя аргумента должно быть `state` (или совпадать с ключом в data). Если назвать его `fsm` — aiogram не найдёт нужную зависимость и выбросит ошибку.

## Приём данных

Фильтр по состоянию срабатывает только тогда, когда чат находится в этом состоянии.

```python
@router.message(Registration.waiting_name)
async def process_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Какая у тебя группа?")
    await state.set_state(Registration.waiting_group)
```

`state.update_data()` сохраняет данные в FSM. В отличие от состояний, данные не переключаются между шагами — они накапливаются.

**Важно:** фильтр по состоянию работает как обычный фильтр. Если пользователь не в состоянии `Registration.waiting_name`, этот хендлер не сработает, даже если сообщение пришло от того же пользователя.

## Полный пример анкеты

Соберём всё вместе — регистрация с тремя полями.

```python
from aiogram import Router, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message

router = Router()

class Registration(StatesGroup):
    waiting_name = State()
    waiting_group = State()
    waiting_email = State()

@router.message(Command("register"))
async def start_registration(message: Message, state: FSMContext):
    await state.clear()  # на случай, если пользователь начал заново
    await message.answer("Как тебя зовут?")
    await state.set_state(Registration.waiting_name)

@router.message(Registration.waiting_name)
async def process_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Какая у тебя группа?")
    await state.set_state(Registration.waiting_group)

@router.message(Registration.waiting_group)
async def process_group(message: Message, state: FSMContext):
    await state.update_data(group=message.text)
    await message.answer("Укажи email:")
    await state.set_state(Registration.waiting_email)

@router.message(Registration.waiting_email)
async def process_email(message: Message, state: FSMContext, db):
    await state.update_data(email=message.text)
    data = await state.get_data()

    # Сохраняем в базу
    await db.execute(
        """
        INSERT INTO users (telegram_id, full_name, group_name, email)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (telegram_id)
        DO UPDATE SET full_name = $2, group_name = $3, email = $4
        """,
        message.from_user.id,
        data["name"],
        data["group"],
        data["email"],
    )

    await message.answer(f"Спасибо, {data['name']}! Регистрация завершена.")
    await state.clear()
```

Обратите внимание на несколько моментов:

1. В `start_registration` вызывается `state.clear()`. Это защита от случая, когда пользователь уже был в середине анкеты и вызвал `/register` заново.
2. Данные накапливаются через `update_data`, а читаются разом через `get_data()`.
3. `state.clear()` в конце — обязательно. Иначе состояние и данные останутся висеть в хранилище.

## Чтение данных

Данные из FSM читаются двумя способами.

**Через `get_data()` — получить всё сразу:**

```python
data = await state.get_data()
name = data.get("name")
group = data.get("group")
```

**Через отдельные ключи — если вам нужно одно поле:**

```python
data = await state.get_data()
email = data["email"]  # KeyError, если поля нет
email = data.get("email")  # None, если поля нет
```

Первый вариант удобнее, если данных много. Второй — если нужно одно-два поля.

## Проверка текущего состояния

Иногда полезно узнать, в каком состоянии находится пользователь — например, чтобы понять, начинать сценарий заново или продолжить.

```python
current = await state.get_state()
if current == Registration.waiting_name.state:
    await message.answer("Ты уже в процессе регистрации")
```

`get_state()` возвращает строку вида `"Registration:waiting_name"` или `None`, если пользователь не в состоянии.

Сравнивать с `Registration.waiting_name` напрямую нельзя — это объект. Сравнивать нужно со строкой:

```python
if current == Registration.waiting_name.state:  # ок
```

Либо через `==` со строкой вручную:

```python
if current == "Registration:waiting_name":  # работает, но менее надёжно
```

Первый вариант предпочтительнее: при рефакторинге класса имя обновится автоматически.

## Выход из состояния

Пользователь может уйти в другой сценарий, не завершив текущий. Тогда его старое состояние помешает новому диалогу. Классическое решение — команда `/cancel`.

```python
@router.message(Command("cancel"))
async def cmd_cancel(message: Message, state: FSMContext):
    current = await state.get_state()
    if current is None:
        await message.answer("Нечего отменять.")
        return
    await state.clear()
    await message.answer("Действие отменено.")
```

Добавьте эту команду во все сценарии — пользователь скажет спасибо.

Ещё один полезный приём — «глобальный выход». В некоторых ботах на любое сообщение вне сценария бот отвечает подсказкой. Это тоже делается через FSM, но требует аккуратной настройки фильтров.

## Хранилище состояний

По умолчанию FSM использует `MemoryStorage` — состояния хранятся в оперативной памяти. Это работает, но данные теряются при перезапуске. Пользователи, которые были в середине диалога, окажутся в подвешенном состоянии: бот не знает, чего он от них ждёт.

Для продакшена используется `RedisStorage` — состояния хранятся в Redis и переживают перезапуск.

```python
from aiogram.fsm.storage.redis import RedisStorage

storage = RedisStorage.from_url("redis://localhost:6379/0")
dp = Dispatcher(storage=storage)
```

Что даёт Redis:

- **Переживает рестарты.** Обновили бота — пользователи остались в своих состояниях.
- **Работает с несколькими экземплярами.** Если запущено несколько копий бота, они видят общее хранилище.
- **Не растёт бесконечно.** TTL можно настроить, старые состояния сами удаляются.

Минимальная настройка Redis через Docker:

```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

И в `main.py`:

```python
from aiogram.fsm.storage.redis import RedisStorage

async def main():
    storage = RedisStorage.from_url("redis://localhost:6379/0")
    dp = Dispatcher(storage=storage)
    # ...
```

Если Redis временно недоступен — бот не запустится. Проверьте, что контейнер работает, прежде чем запускать бота.

## Советы

**Не храните в FSM большие объёмы данных.** Это не база. FSM — для временных данных, нужных только во время диалога. Картинки, файлы, длинные тексты — в файловую систему или БД, а в FSM только ссылку.

**Всегда вызывайте `state.clear()` после завершения.** Иначе данные останутся висеть и займут память. В MemoryStorage они уйдут только при рестарте, в Redis — по TTL.

**Продумывайте выход из состояния.** Если пользователь уйдёт в другой сценарий, его старое состояние может помешать. Команда `/cancel` — обязательный минимум.

**Не путайте `update_data` и `set_state`.** `set_state` переключает шаг диалога, `update_data` накапливает данные. Это разные вещи, хотя используются вместе.

**Валидируйте данные.** Пользователь может ввести что угодно. Если ждёте email — проверьте, что это email. Если ждёте число — попробуйте преобразовать. FSM не защищает от мусора:

```python
@router.message(Registration.waiting_email)
async def process_email(message: Message, state: FSMContext):
    email = message.text.strip()
    if "@" not in email or "." not in email:
        await message.answer("Это не похоже на email. Попробуй ещё раз.")
        return  # остаёмся в том же состоянии
    await state.update_data(email=email)
    # ...
```

Для отладки FSM удобно логировать текущее состояние в каждом хендлере:

```python
import logging

@router.message(Registration.waiting_name)
async def process_name(message: Message, state: FSMContext):
    logging.info(f"User {message.from_user.id} in state {await state.get_state()}")
    # ...
```

Если бот «теряет» пользователя посреди сценария — чаще всего виноват `state.clear()` где-то не там, либо данные не сохранились из-за ошибки в `update_data`. Логи помогут найти виновника.
