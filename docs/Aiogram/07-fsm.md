# 7. Машина состояний FSM

FSM (Finite State Machine, конечный автомат) — это способ вести с пользователем пошаговый диалог, где на каждом шаге бот ждёт определённого ответа.

## Зачем нужна FSM

Представим, что бот запрашивает у пользователя анкету: имя, группу, email. Без FSM бот не понимает, что именно сейчас от него хотят. Пользователь отправит сообщение «Иван», и бот не знает — это имя, это группа или это случайный текст.

FSM решает эту проблему: бот помнит, на каком шаге находится каждый пользователь. Пока пользователь в состоянии «ожидаю имя» — любое сообщение считается именем. Пока в состоянии «ожидаю группу» — считается группой.

## Состояния

Состояния описываются через класс-наследник `StatesGroup`. Каждое поле — это отдельное состояние.

```python
from aiogram.fsm.state import State, StatesGroup

class Registration(StatesGroup):
    waiting_name = State()
    waiting_group = State()
    waiting_email = State()
```

Обычно класс называется по смыслу сценария, а состояния — по тому, чего бот ждёт от пользователя.

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

## Завершение

Когда все данные собраны, можно их прочитать через `state.get_data()`, сохранить в базу и очистить состояние.

```python
@router.message(Registration.waiting_email)
async def process_email(message: Message, state: FSMContext):
    await state.update_data(email=message.text)
    data = await state.get_data()
    # ... сохраняем data в базу
    await message.answer(f"Спасибо, {data['name']}!")
    await state.clear()
```

## Хранилище состояний

По умолчанию FSM использует `MemoryStorage` — состояния хранятся в оперативной памяти. Это работает, но данные теряются при перезапуске. Пользователи, которые были в середине диалога, окажутся в подвешенном состоянии: бот не знает, чего он от них ждёт.

Для продакшена используется `RedisStorage` — состояния хранятся в Redis и переживают перезапуск.

```python
from aiogram.fsm.storage.redis import RedisStorage

storage = RedisStorage.from_url("redis://localhost:6379/0")
dp = Dispatcher(storage=storage)
```

## Советы

Не храните в FSM большие объёмы данных — это не база. FSM — для временных данных, нужных только во время диалога.

Используйте `state.clear()` после завершения — иначе данные останутся висеть и займут память.

Продумывайте выход из состояния. Если пользователь уйдёт в другой сценарий, его старое состояние может помешать. Иногда стоит добавлять команду `/cancel`, которая сбрасывает любое состояние.
