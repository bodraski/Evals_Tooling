# Черновик PR: автоопределение reasoning parser и ограничение fallback-парсинга тегов по модели

# https://github.com/UKGovernmentBEIS/inspect_ai/compare/main...gornkv:inspect_ai:autodetect_reasoning_parser

## Контекст

Inspect поддерживает OpenAI-compatible providers и локальные серверы вроде vLLM и SGLang. Некоторые reasoning-модели возвращают приватный reasoning в структурированных API-полях, например `reasoning_content`, `reasoning` или `reasoning_details`. В других конфигурациях reasoning возвращается внутри обычного текста assistant-сообщения через теги:

```text
<think>private reasoning</think>final answer
```

Существующий fallback parser извлекает такие теги из `content` и превращает их в `ContentReasoning`. Это полезно, когда сервер сам не выполнил parsing reasoning.

Проблема в том, что отсутствие структурированных reasoning fields само по себе не доказывает, что `<think>` parsing нужно запускать для любой модели.

## Механизм проблемы

Есть два отдельных режима отказа.

### 1. Серверный parser отсутствует или отключен

Для модели вроде Qwen3, запущенной через vLLM, reasoning может приходить как tagged content, если vLLM не был стартован с подходящим reasoning parser:

```bash
inspect eval math.py --model vllm/Qwen/Qwen3-8B -M reasoning_parser=qwen3
```

Если сервер не настроен с `reasoning_parser=qwen3`, OpenAI-compatible response может не содержать структурированного reasoning field. Вместо этого reasoning text может оказаться внутри `content` как `<think>...</think>`.

В таком случае fallback parser Inspect полезен: он восстанавливает reasoning из content и удаляет его из видимого ответа.

### 2. Модель без tag-based reasoning возвращает literal `<think>` text

Для модели, про которую неизвестно, что она использует `<think>` tags как reasoning protocol, тот же fallback может быть вредным. Если обычный assistant answer содержит literal text вроде:

```text
Use the tag <think>...</think> in this template.
```

или пример кода с такими тегами, безусловный fallback parsing может ошибочно удалить часть видимого ответа и сохранить ее как private reasoning.

Это особенно рискованно для OpenAI-compatible providers, потому что одна только форма API неоднозначна. Пустое поле `reasoning` может означать:

- сервер забыл распарсить tag-based reasoning model;
- модель не использует tag-based reasoning;
- модель не делала reasoning в этом response;
- provider использует другой structured mechanism;
- response просто содержит обычный текст с tag-like syntax.

Поэтому fallback нужно ограничить моделями, от которых ожидаются reasoning tags, сохранив старое поведение для случаев, когда имя модели недоступно.

## Изменение

Изменение добавляет mapping из модели в parser в `src/inspect_ai/model/_reasoning.py`:

```python
REASONING_PARSER_BY_MODEL: tuple[tuple[str, str], ...] = (
    (r"(^|/)Qwen3(?:[-._/]|$)[^/]*", "qwen3"),
    (r"(^|/)QwQ-32B(?:[-._/]|$)[^/]*", "deepseek_r1"),
    (r"(^|/)DeepSeek-R1(?:[-._/]|$)[^/]*", "deepseek_r1"),
    (r"(^|/)DeepSeek-V3\.1(?:[-._/]|$)[^/]*", "deepseek_v3"),
    (r"(^|/)GLM-4\.5(?:[-._/]|$)[^/]*", "glm45"),
    (r"(^|/)granite-3\.2(?:[-._/]|$)[^/]*", "granite"),
    (r"(^|/)MiniMax-M2(?:[-._/]|$)[^/]*", "minimax_m2_append_think"),
)
```

и helper для поиска:

```python
def reasoning_parser_for_model(model: str | None) -> str | None:
    if not model:
        return None

    for pattern, parser in REASONING_PARSER_BY_MODEL:
        if re.search(pattern, model, re.IGNORECASE):
            return parser

    return None
```

Regex'ы намеренно написаны так, чтобы матчить типичные Hugging Face names и custom quantized variants, например `Qwen/Qwen3-8B` и `unsloth/Qwen3-8B-AWQ`, но не быть слишком широкими для вероятных следующих поколений семейств вроде `Qwen4` или `GLM-5`.

## Поведение fallback-парсинга

`parse_content_with_reasoning()` теперь принимает необязательное имя модели:

```python
def parse_content_with_reasoning(
    content: str, model: str | None = None
) -> tuple[str, ReasoningCapsule | None]:
```

Если `model` передан и не соответствует известным tag-based families, функция возвращает content без изменений:

```python
if model and not reasoning_parser_for_model(model):
    return content, None
```

Если `model` не передан, старое поведение сохраняется: функция все равно ищет `<think>` tags. Это важно для существующих call sites и logs, где имя модели может быть недоступно.

OpenAI-compatible путь parsing теперь прокидывает модель в fallback parser:

- `messages_from_openai()` передает `model` при parsing assistant string content.
- `content_from_openai()` принимает `model` и передает его в `parse_content_with_reasoning()`.
- `chat_message_assistant_from_openai()` передает `model` в `parse_reasoning_content()`.
- `parse_reasoning_content()` принимает `model` и ищет `<think>` tags только если модель не указана или попала в mapping.

Явные API reasoning fields по-прежнему принимаются для любой модели. Проверка по имени модели применяется только к tag fallback, но не к `reasoning_content`, `reasoning` или `reasoning_details`.

## Предотвращение дублирующего извлечения reasoning

`messages_from_openai()` уже парсит `<think>` tags из assistant `content` до проверки explicit reasoning fields. После добавления model-aware parsing в `parse_reasoning_content()` второй вызов мог бы повторно распарсить тот же исходный `message["content"]`.

Проблемная последовательность выглядела бы так:

1. `messages_from_openai()` читает `message["content"]`.
2. Вызывает `parse_content_with_reasoning()` и извлекает `<think>private</think>` в `smuggled_reasoning`.
3. Собирает visible content из очищенного текста.
4. Затем вызывает `parse_reasoning_content()` на исходном message object.
5. Если `parse_reasoning_content()` тоже ищет tags, он снова видит исходный `<think>` text и возвращает второй reasoning item.

В этом пути `messages_from_openai()` нет дедупликации. Похожая обработка есть в другом месте для OpenAI Responses content, но здесь она не применяется. Поэтому второй вызов теперь отключает tag lookup и проверяет только structured fields:

```python
parse_result = parse_reasoning_content(
    message, model, look_for_reasoning_tags=False
)
```

Это сохраняет нужное двухшаговое поведение:

- первый шаг: распарсить smuggled `<think>` reasoning из content;
- второй шаг: дополнительно извлечь explicit API reasoning fields, если они есть.

## Поведение локального vLLM server

Когда Inspect запускает локальный vLLM server, он теперь заполняет `reasoning_parser`, если пользователь не передал его явно и модель распознана:

```python
server_args = dict(self.server_args)
if "reasoning_parser" not in server_args:
    reasoning_parser = reasoning_parser_for_model(model_path)
    if reasoning_parser:
        server_args["reasoning_parser"] = reasoning_parser
```

Это сохраняет приоритет явной пользовательской конфигурации. Если пользователь передал `-M reasoning_parser=...`, Inspect не перезаписывает это значение. Если модель неизвестна, Inspect не угадывает.

Для типичного случая:

```bash
inspect eval math.py --model vllm/Qwen/Qwen3-8B
```

Inspect теперь может стартовать vLLM с тем же parser, который пользователь иначе должен был бы указать вручную:

```bash
-M reasoning_parser=qwen3
```

Практический эффект в том, что vLLM может возвращать structured reasoning fields, если поддерживает этот parser, и необходимость client-side fallback parsing уменьшается.

## Поведение локального SGLang server

Для SGLang изменение намеренно другое. Вместо model mapping Inspect явно устанавливает:

```python
if "reasoning_parser" not in self.server_args:
    self.server_args["reasoning_parser"] = "auto"
```

У SGLang есть режим `auto` для reasoning parser, поэтому Inspect не нужно дублировать определение семейства модели на своей стороне. Явная пользовательская конфигурация остается приоритетной: если `reasoning_parser` уже есть в server args, Inspect оставляет его без изменений.

## Документация

`docs/reasoning.qmd` обновлен и описывает новое поведение локальных серверов:

- vLLM parser selection выполняется автоматически для известных семейств моделей и по-прежнему может быть переопределен через `-M`.
- SGLang стартует с `reasoning_parser=auto`, если пользователь не передал parser явно.

## Почему выбрано такое решение

Выбранный подход объединяет две защиты:

1. Корректно конфигурировать local servers, когда Inspect сам управляет server startup.
2. Ограничить client-side tag fallback семействами моделей, от которых ожидается tag-based reasoning.

Это закрывает обе стороны проблемы. Корректная server configuration уменьшает число responses, где reasoning приходит smuggled в `content`; проверка по имени модели предотвращает случайное удаление нормального answer text для нерелевантных моделей.

Mapping размещен в `_reasoning.py`, потому что он используется и fallback parser, и кодом запуска local provider. Хранение mapping рядом с `parse_content_with_reasoning()` также делает правило видимым ровно на границе, где текст превращается в `ReasoningCapsule`.

## Рассмотренные альтернативы

### Опираться только на пустые structured reasoning fields

Это был исходный ориентир, но он слишком неоднозначен. Пустой `reasoning` не доказывает, что `<think>` tags семантически являются private reasoning. Он доказывает только то, что provider не вернул structured reasoning для данного response.

Использование одной только пустоты поля может испортить normal content для моделей, которые упоминают `<think>` буквально.

### Запрашивать конфигурацию vLLM server перед parsing

Теоретически Inspect мог бы читать startup settings vLLM или server metadata и решать, включен ли `reasoning_parser`. Это полезно только когда Inspect контролирует local server или сервер предоставляет надежную metadata.

Это не решает remote OpenAI-compatible endpoints, manually started servers, proxies или SGLang auto mode. Также появляется operational coupling: решение о parser зависело бы от provider-specific introspection, а не от response плюс model identity.

Текущая реализация все равно использует server settings там, где они полезны: когда Inspect запускает vLLM, он прямо устанавливает `reasoning_parser`. Для parsing responses имя модели является более простым и более portable guard.

### Всегда конфигурировать vLLM и удалить client-side fallback

Это работало бы только для local vLLM servers, запущенных Inspect, и только для моделей с поддерживаемыми vLLM parsers. Такой подход не покрывает существующие logs, remote compatible endpoints, manually started servers или случаи, когда provider все равно возвращает tagged content.

Fallback остается необходимым, но теперь он ограничен проверкой.

### Использовать SGLang model mapping вместо `auto`

SGLang уже поддерживает automatic parser selection. Дублировать model mapping в Inspect было бы более хрупко, чем попросить SGLang использовать собственный parser detection. Поэтому изменение устанавливает `reasoning_parser=auto` только когда пользователь не передал значение.

### Матчить имена моделей очень широко

Широкие regex'ы вроде `Qwen` или `DeepSeek` ловили бы слишком много нерелевантных или будущих моделей. Это могло бы назначить parser модели следующего поколения, у которой reasoning protocol изменился. Поэтому mapping нацелен на известные reasoning families и versions, но допускает типичные suffixes для quantization и custom packaging.

## Тесты

Изменение добавляет focused tests в существующие test files:

- `tests/model/test_reasoning_parse.py`
  - mapped Hugging Face и quantized model names резолвятся в ожидаемый parser;
  - вероятные имена моделей следующего поколения не матчятся;
  - unmapped models не запускают `<think>` fallback;
  - отсутствие model name сохраняет старое tag fallback behavior.

- `tests/model/test_openai_chat_messages.py`
  - non-tag model names не парсят `<think>` из normal content;
  - mapped model names парсят `<think>`;
  - explicit `reasoning_content` продолжает работать даже для non-tag model name.

Локально были выполнены дополнительные проверки:

```bash
uv run pytest tests/model/test_reasoning_parse.py tests/model/test_openai_chat_messages.py
uv run pytest tests/model
uv run ruff check src/inspect_ai/model/_reasoning.py src/inspect_ai/model/_openai.py src/inspect_ai/model/_providers/vllm.py src/inspect_ai/model/_providers/sglang.py tests/model/test_reasoning_parse.py tests/model/test_openai_chat_messages.py
git diff --check
```

Также был запущен полный `uv run pytest`. Он прошел область model/provider, которую затрагивает это изменение, а затем долго выполнял unrelated infrastructure step, который вызвал `docker compose pull` из startup test. Пользователь попросил ждать, затем прервал turn до завершения полного suite, поэтому финального результата полного suite нет.

## Риск

Главный функциональный риск — false negatives в model mapping. Custom model, которая действительно использует `<think>` tags, но не матчится известными regex'ами, больше не будет парсить эти tags, если имя модели передано. Это намеренное решение: безопаснее не вырезать visible answer content для неизвестных моделей, чем считать каждую строку `<think>` private reasoning.

Пользователи все еще могут получить корректное поведение через server-side parser configuration, а mapping можно расширять по мере валидации новых известных tag-based model families.
