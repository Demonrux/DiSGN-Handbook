# 12. Форматирование текста и отправка медиа

До этого мы отправляли только простой текст. Но Telegram умеет гораздо больше: жирный, курсив, ссылки, код, цитаты, спойлеры, фото, видео, документы, голосовые, стикеры. Разберёмся, как этим пользоваться.

## Режимы разметки

В Telegram два режима разметки: **HTML** и **MarkdownV2**. Выбрать можно через `parse_mode`.

```python
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

bot = Bot(
    token="...",
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
```

Настройка `default` применяется ко **всем** сообщениям. Не нужно каждый раз передавать `parse_mode` в `send_message` — это удобно и защищает от забывчивости.

### Почему HTML

Из двух режимов я рекомендую HTML. Причины:

- **Проще экранирование.** В MarkdownV2 нужно экранировать почти все специальные символы (`_`, `*`, `[`, `]`, `(`, `)`, `~`, `` ` ``, `>`, `#`, `+`, `-`, `=`, `|`, `{`, `}`, `.`, `!`). В HTML — только `<`, `>`, `&`.
- **Понятнее синтаксис.** `<b>жирный</b>` читается легче, чем `*жирный*`.
- **Меньше багов.** MarkdownV2 ломается от любой неэкранированной точки или восклицательного знака.

Дальше все примеры — на HTML.

## Поддерживаемые теги

Вот что умеет HTML-разметка Telegram:

| Тег | Результат |
|---|---|
| `<b>текст</b>` | **жирный** |
| `<i>текст</i>` | *курсив* |
| `<u>текст</u>` | подчёркнутый |
| `<s>текст</s>` | зачёркнутый |
| `<code>текст</code>` | `моноширинный` |
| `<pre>текст</pre>` | блок кода |
| `<a href="url">текст</a>` | ссылка |
| `<tg-spoiler>текст</tg-spoiler>` | спойлер |
| `<blockquote>текст</blockquote>` | цитата |
| `<tg-emoji emoji-id="...">🎉</tg-emoji>` | кастомный эмодзи |

Комбинировать можно как угодно — теги вкладываются:

```python
await message.answer(
    "<b>Жирный</b>, <i>курсив</i>, <u>подчёркнутый</u>, "
    "<s>зачёркнутый</s>, <code>моно</code>\n\n"
    'Ссылка: <a href="https://example.com">example.com</a>\n'
    "<tg-spoiler>Секрет</tg-spoiler>"
)
```

Блок кода — для многострочных фрагментов:

```python
code = "def hello():\n    print('world')"
await message.answer(f"<pre>{code}</pre>")
```

### Экранирование

Если пользователь введёт `<` или `>` — Telegram попытается распарсить их как тег и вернёт ошибку `Bad Request: can't parse entities`. Чтобы этого избежать, используйте `html.escape`:

```python
import html

@router.message(F.text)
async def echo(message: Message):
    safe_text = html.escape(message.text)
    await message.answer(f"Ты написал: <b>{safe_text}</b>")
```

Или есть встроенный хелпер:

```python
from aiogram.utils.markdown import hbold, hitalic, hlink, hcode, hspoiler

await message.answer(
    hbold("Жирный") + ", " + hitalic("курсив") + ", " +
    hlink("ссылка", "https://example.com")
)
```

Эти функции экранируют содержимое за вас. Для критичных мест — безопаснее.

## Отправка медиа

Все методы отправки медиа принимают `caption` — подпись к файлу. Подпись, как и текст, поддерживает HTML-разметку.

### Фото

```python
from aiogram.types import FSInputFile

@router.message(Command("cat"))
async def cmd_cat(message: Message):
    photo = FSInputFile("images/cat.jpg")
    await message.answer_photo(
        photo,
        caption="<b>Котик</b> загружен",
    )
```

`FSInputFile` — обёртка над файлом в файловой системе. Есть ещё:

- **`URLInputFile`** — если файл лежит по URL.
- **`BufferedInputFile`** — если у вас есть байты в памяти.
- **Строка `file_id`** — если файл уже отправлялся и Telegram его знает.

`file_id` — самый быстрый способ. Когда вы отправляете фото в первый раз, Telegram возвращает `file_id`, и при повторной отправке можно использовать его вместо файла:

```python
@router.message(Command("meme"))
async def cmd_meme(message: Message):
    # file_id, полученный когда-то давно
    await message.answer_photo("AgACAgIAAxkBAAIC...")
```

### Документ

```python
await message.answer_document(
    FSInputFile("reports/2026.pdf"),
    caption="Отчёт за 2026 год",
)
```

Документ — любой файл, который Telegram не сжимает. В отличие от фото, здесь можно отправить PDF, ZIP, XLSX и всё остальное.

### Видео, аудио, голосовое

```python
await message.answer_video(FSInputFile("video.mp4"))
await message.answer_audio(FSInputFile("song.mp3"), title="Название")
await message.answer_voice(FSInputFile("voice.ogg"))
```

Голосовое (`answer_voice`) — это то, что в Telegram отображается как «кружочек» с волной.

### Альбом (media group)

Для отправки нескольких файлов одним сообщением используется `MediaGroupBuilder`:

```python
from aiogram.utils.media_group import MediaGroupBuilder

@router.message(Command("album"))
async def cmd_album(message: Message):
    builder = MediaGroupBuilder(caption="Мой альбом")
    builder.add_photo(media=FSInputFile("1.jpg"))
    builder.add_photo(media=FSInputFile("2.jpg"))
    builder.add_photo(media=FSInputFile("3.jpg"))
    await message.answer_media_group(builder.build())
```

Альбом отправится как одно сообщение с несколькими вложениями. Подпись (`caption`) у альбома одна — прикрепляется к первому файлу.

## Скачивание файлов

Если пользователь прислал файл, его можно скачать. Путь: получить `file_id`, запросить у Telegram информацию о файле, скачать по URL.

```python
from aiogram import Bot
from aiogram.types import Message, File

@router.message(F.document)
async def handle_document(message: Message, bot: Bot):
    file_id = message.document.file_id
    file: File = await bot.get_file(file_id)

    # Скачиваем
    destination = f"downloads/{message.document.file_name}"
    await bot.download_file(file.file_path, destination)
    await message.answer(f"Файл сохранён: {destination}")
```

Есть удобный хелпер `bot.download()`, который принимает `file_id` или `File` и сразу сохраняет в файл или буфер:

```python
await bot.download(message.document, destination="downloads/file.pdf")
```

**Важно:** у Telegram есть лимит 20 МБ на скачивание через Bot API. Для больших файлов нужен локальный Bot Server.

## Редактирование сообщений

Отправленное сообщение можно редактировать — текст, разметку, клавиатуру.

```python
@router.message(Command("edit"))
async def cmd_edit(message: Message):
    sent = await message.answer("Исходный текст")
    await asyncio.sleep(2)
    await sent.edit_text("Обновлённый текст")
```

Полезные методы:

- `edit_text(text, reply_markup=...)` — заменить текст и/или клавиатуру.
- `edit_reply_markup(reply_markup=...)` — заменить только клавиатуру.
- `edit_caption(...)` — для медиа-сообщений.

**Ошибка `message is not modified`.** Telegram не разрешает редактировать сообщение, если новый текст совпадает со старым. Если вы часто редактируете одно и то же сообщение (например, таймер), проверяйте, изменилось ли что-то:

```python
try:
    await sent.edit_text(new_text)
except TelegramBadRequest as e:
    if "message is not modified" not in str(e):
        raise
```

Или сравнивайте с предыдущим значением сами.

## Удаление сообщений

```python
await message.delete()          # удалить текущее
await sent.delete()             # удалить отправленное ранее
await bot.delete_message(chat_id, message_id)
```

Бот может удалять свои сообщения в личке без ограничений. В группах — только если имеет права администратора.

## Копирование и пересылка

```python
await bot.copy_message(
    chat_id=target_chat,
    from_chat_id=message.chat.id,
    message_id=message.message_id,
)
```

`copy_message` создаёт копию без пометки «Переслано». `forward_message` — с пометкой.

## Работа с реакциями

Бот может ставить реакции на сообщения (Telegram 7.0+):

```python
from aiogram.types import ReactionTypeEmoji

await bot.set_message_reaction(
    chat_id=message.chat.id,
    message_id=message.message_id,
    reaction=[ReactionTypeEmoji(type="emoji", emoji="👍")],
)
```

Это ненавязчивый способ показать, что бот «принял» сообщение — без отправки отдельного ответа.

## Индикатор «печатает»

Пока бот думает, полезно показать пользователю, что он работает. Для этого есть `send_chat_action`:

```python
@router.message(Command("slow"))
async def cmd_slow(message: Message, bot: Bot):
    await bot.send_chat_action(message.chat.id, "typing")
    await asyncio.sleep(3)
    await message.answer("Готово!")
```

Типы действий: `typing`, `upload_photo`, `record_video`, `upload_document`, `find_location`, `record_voice` и другие. Индикатор живёт 5 секунд — если операция дольше, отправляйте повторно.

## Совет

Разметка Telegram ограничена. Не пытайтесь впихнуть в сообщение Markdown-таблицу, вложенные списки или что-то похожее на HTML-вёрстку. Если нужна сложная структура — используйте `WebApp` (мини-приложение) или отправляйте документ.

И второе: не забывайте про `html.escape()` для пользовательского ввода. Иначе пользователь отправит `<b>ха-ха</b>` и либо получит ошибку от Telegram, либо его тег «просочится» в разметку.
