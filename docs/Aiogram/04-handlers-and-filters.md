# 4. Хендлеры и фильтры

Хендлер — это функция, которая вызывается в ответ на событие. Фильтр — это условие, при котором хендлер срабатывает. Вместе они образуют основу всей логики бота.

## Что такое хендлер

Хендлер — это обычная асинхронная функция. Она принимает объект события (например, `Message`) и делает что-то полезное.

```python
from aiogram import Router
from aiogram.filters import CommandStart
from aiogram.types import Message

router = Router()

@router.message(CommandStart())
async def cmd_start(message: Message):
    await message.answer("Привет!")
```

Декоратор `@router.message(...)` регистрирует функцию как обработчик сообщений. Аргументы декоратора — это фильтры. Функция вызывается только тогда, когда все фильтры совпали.

## Виды событий

aiogram поддерживает много типов событий. Основные:

- `message` — пришло сообщение (текст, фото, видео, документ и т.д.).
- `callback_query` — нажата inline-кнопка.
- `inline_query` — пользователь вызывает бота через `@username` в любом чате.
- `chat_member` — изменился статус участника чата.
- `my_chat_member` — изменился статус бота в чате.
- `poll_answer` — пользователь проголосовал в опросе.
- `pre_checkout_query` — этап подтверждения платежа.
- `shipping_query` — запрос вариантов доставки.

У каждого типа — свой набор полей и свои фильтры. Например, у `callback_query` есть `data`, а у `message` — `text`.

## Первый аргумент — всегда событие

Первый аргумент функции — это объект события. Через него можно получить данные и ответить.

```python
@router.message()
async def echo(message: Message):
    await message.answer(f"Ты написал: {message.text}")
```

Объект `Message` содержит всё: `from_user` (кто написал), `chat` (куда написал), `text` (что написал), `date` (когда). Если сообщение содержит фото — `message.photo`, документ — `message.document`, и так далее.

Найти нужное поле можно всегда по документации: [docs.aiogram.dev](https://docs.aiogram.dev/en/latest/api/types/message.html).

## Помимо события — другие зависимости

aiogram сам подставляет в хендлер всё, что нужно, если имя аргумента совпадает с ключом в `data`. Это работает через **dependency injection** — ту самую инъекцию зависимостей, о которой мы поговорим в главе про мидлвари.

Помимо `message` в функцию можно попросить:

- `state: FSMContext` — контекст машины состояний (глава 7).
- `bot: Bot` — объект бота, если нужно отправить сообщение в другой чат.
- `db` — что угодно, что положила мидлварь (глава 8).
- Свои кастомные зависимости.

```python
@router.message(Command("secret"))
async def cmd_secret(message: Message, bot: Bot, state: FSMContext):
    await message.answer("Секретное сообщение")
    await bot.send_message(chat_id=123456789, text="Кто-то вызвал /secret")
    await state.clear()
```

## Фильтры

Фильтр — это условие, которое должно выполниться, чтобы хендлер сработал. Фильтры бывают встроенные и кастомные.

### Встроенные фильтры

**CommandStart** — срабатывает на команду `/start`. Отдельный фильтр, потому что это стандартная команда для всех ботов.

**Command** — срабатывает на любую указанную команду.

```python
from aiogram.filters import Command

@router.message(Command("help"))
async def cmd_help(message: Message):
    await message.answer("Я умею многое")
```

Команда указывается без слэша. Можно передать несколько команд:

```python
@router.message(Command("help", "about"))
```

А можно и добавить префикс — если хотите поддерживать команды с восклицательным знаком:

```python
@router.message(Command("help", prefix="!/"))
```

**CommandObject** — если нужно достать аргументы команды:

```python
from aiogram.filters import CommandObject

@router.message(Command("echo"))
async def cmd_echo(message: Message, command: CommandObject):
    # Пользователь ввёл "/echo привет мир"
    args = command.args  # "привет мир"
    await message.answer(args or "Пусто")
```

Это удобно для команд вроде `/ban 12345` или `/note купить хлеб`.

### Как работает проверка фильтров

Когда приходит событие, диспетчер проходит по всем зарегистрированным хендлерам по порядку. Для каждого хендлера вызываются его фильтры. Если все вернули `True` — хендлер вызывается, и проверка останавливается. Если хотя бы один вернул `False` — проверяется следующий хендлер.

Схематично:

```
Update
  │
  ▼
Handler 1: [filter A] → False
  │
  ▼
Handler 2: [filter B] → True, [filter C] → True  ──► ВЫЗОВ
  │
  ▼
Handler 3: не проверяется
```

Это значит, что **порядок регистрации имеет значение**. Если у вас есть общий хендлер `@router.message()` без фильтров, зарегистрированный первым, он перехватит все сообщения, и до остальных дело не дойдёт. Общие хендлеры регистрируются последними.

Классическая ловушка:

```python
# Неправильно: catch_all перехватит всё
@router.message()
async def catch_all(message: Message):
    await message.answer("Я перехватил всё!")

@router.message(Command("help"))
async def cmd_help(message: Message):
    await message.answer("Помощь")  # никогда не вызовется
```

Правильно — сначала специфичные, потом общие:

```python
# Правильно
@router.message(Command("help"))
async def cmd_help(message: Message):
    await message.answer("Помощь")

@router.message()
async def catch_all(message: Message):
    await message.answer("Я не понял команду")
```

## Ответ пользователю

Самый частый способ ответить — `message.answer()`. Это эквивалент `bot.send_message(chat_id=message.chat.id, ...)`, только без необходимости указывать чат: он уже известен из сообщения.

```python
await message.answer("Привет!")
```

Кроме `answer()` у сообщения есть ещё несколько методов:

- `message.reply(...)` — ответить с цитированием исходного сообщения.
- `message.edit_text(...)` — изменить текст (только для сообщений, отправленных ботом).
- `message.delete()` — удалить сообщение.
- `message.answer_photo(...)`, `answer_video(...)`, `answer_document(...)` — отправить медиа.

```python
@router.message(Command("del"))
async def cmd_del(message: Message):
    await message.delete()
    await message.answer("Сообщение удалено")
```

Для нажатий на кнопки есть `callback.answer()` — он убирает «часики» на кнопке и, если нужно, показывает всплывающее уведомление.

```python
from aiogram.types import CallbackQuery

@router.callback_query()
async def on_button(callback: CallbackQuery):
    await callback.answer("Кнопка нажата")
    await callback.message.answer("Ответ в чат")
```

**Важно:** `callback.answer()` нужно вызывать **всегда**, даже если вам нечего ответить. Иначе у пользователя на кнопке будут вечно висеть «часики» — Telegram ждёт подтверждения, что нажатие обработано.

```python
@router.callback_query(F.data == "noop")
async def noop(callback: CallbackQuery):
    await callback.answer()  # без текста — просто убрать часики
```

## Кастомные фильтры

Встроенных фильтров хватает не всегда. Например, вы хотите пропускать к хендлеру только админов. Можно написать собственный фильтр.

Фильтр — это класс, наследник `BaseFilter`, с методом `__call__`, который возвращает `True` или `False`. Метод может быть асинхронным.

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
@router.message(IsAdmin(admin_ids=[123, 456]), Command("ban"))
async def cmd_ban(message: Message):
    await message.answer("Бан выполнен")
```

Фильтр принимает объект события и любые данные, которые передал aiogram. Через `data` можно достать зависимости, добавленные мидлварями:

```python
class IsPremium(BaseFilter):
    async def __call__(self, message: Message, db) -> bool:
        user = await db.fetchrow(
            "SELECT is_premium FROM users WHERE telegram_id = $1",
            message.from_user.id,
        )
        return bool(user and user["is_premium"])
```

Имя аргумента `db` должно совпадать с ключом, который положила мидлварь. Это та же механика, что и в хендлерах.

**Когда писать кастомный фильтр, а когда использовать F?** Правило простое: если условие влезает в одну строку и не требует запросов к внешним сервисам — берите `F` (глава 5). Если нужна логика сложнее (проверка в БД, вызов API, вычисления) — пишите класс.

## Фильтр на уровне роутера

Иногда нужно, чтобы фильтр применялся ко **всем** хендлерам внутри роутера. Например, роутер `admin_router` — только для админов. Можно повесить фильтр на сам роутер:

```python
admin_router = Router()
admin_router.message.filter(IsAdmin(admin_ids=[123, 456]))
admin_router.callback_query.filter(IsAdmin(admin_ids=[123, 456]))
```

Теперь любой хендлер внутри `admin_router` получит событие только если пользователь — админ. Писать фильтр в каждом декораторе не нужно.

## Порядок и приоритет

Полный порядок обработки Update выглядит так:

1. Update попадает в `Dispatcher`.
2. Пробегаем роутеры в порядке подключения через `dp.include_router(...)`.
3. Внутри каждого роутера — хендлеры в порядке регистрации.
4. Для каждого хендлера сначала срабатывают фильтры на роутере, потом на декораторе.
5. Первый хендлер, у которого **все** фильтры вернули `True`, вызывается. Остальные не проверяются.

Если ни один хендлер не подошёл, Update просто игнорируется. Ни ошибки, ни предупреждения — по умолчанию aiogram молча его выбрасывает.

## Совет

Если хендлер «не срабатывает», а вы уверены, что фильтры правильные — проверьте три вещи:

1. Не перехватывает ли его другой хендлер, зарегистрированный раньше.
2. Подключён ли роутер через `dp.include_router(...)`.
3. Включён ли `logging` — по логам `aiogram.dispatcher` видно, какие Update'ы приходят и какие хендлеры проверяются.

В 90% случаев проблема в первом пункте.
