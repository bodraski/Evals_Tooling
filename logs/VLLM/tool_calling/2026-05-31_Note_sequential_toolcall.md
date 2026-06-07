# Finding: Qwen3-8B параллельный tool calling ломает sequential задачи

## Модель и конфигурация

- Model: `Qwen/Qwen3-8B-AWQ`
- vLLM флаги: `--enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser qwen3`

## Поведение

Qwen3-8B поддерживает параллельный tool calling — при наличии нескольких инструментов
модель может вызвать их все в одном ответе, не дожидаясь результатов.

Для задачи где второй вызов зависит от результата первого:

```
1. get_secure_vault_code() → возвращает "SECRET-QWEN-9921"
2. unlock_vault(code="SECRET-QWEN-9921") → использует реальный результат
```

Qwen3-8B вызвала оба инструмента одновременно:

```json
"tool_calls": [
  {"name": "get_secure_vault_code", "arguments": "{}"},
  {"name": "unlock_vault", "arguments": "{\"code\": \"7370616e636520726564617465\"}"}
]
```

Код `7370616e636520726564617465` — hex строка придуманная моделью,
так как реального результата первого вызова у неё не было.

## Почему это silent failure

- eval проходит без ошибок
- tool вызван, аргумент передан
- score неверный потому что код неправильный
- никакого предупреждения нет

## Сравнение с Qwen2.5-7B-Instruct

Qwen2.5-7B-Instruct не поддерживает параллельный tool calling —
делает вызовы последовательно и корректно передаёт результат первого во второй.

## Фикс

Отключить параллельный tool calling через параметр запроса:

```python
# в Inspect AI
from inspect_ai.solver import use_tools, generate
from inspect_ai.model import GenerateConfig

solver = [
    use_tools([tool_call_1(), tool_call_2()]),
    generate(
        GenerateConfig(parallel_tool_calls=False)
    )
]
```


## Вывод для документации

При использовании Qwen3 с sequential tool calling (где второй вызов
зависит от результата первого) необходимо явно отключать параллельный режим.
Без этого модель генерирует аргументы для последующих вызовов самостоятельно,
что приводит к неверным результатам без каких-либо ошибок.

**Гипотеза**:
Более сильная модель обычно уменьшает вероятность проблемы, но не гарантирует её отсутствие.

### FOR THE BUGREPORT

Qwen3-8B emits dependent tool calls in a single assistant turn when parallel tool calling is enabled. The model hallucinates arguments for downstream tool calls instead of waiting for upstream tool results. Setting parallel_tool_calls=False eliminates the issue and results in correct sequential tool execution.

For evaluations involving sequential tool dependencies, consider disabling parallel tool calls. Some models may emit multiple dependent tool calls in a single assistant turn and hallucinate arguments for downstream calls instead of waiting for upstream tool results.

## Минимальный пример с двумя зависимыми инструментами.
Лог с parallel_tool_calls=True.   -2026-05-31_18-04_mitm_log.json
Лог с parallel_tool_calls=False.  - 2026-05-31_16-36_mitm_log.json

## Результат на 2-3 моделях

For evaluations involving dependent tool calls, model capability can significantly affect results even when the serving configuration is identical. Smaller models may emit downstream tool calls before upstream tool results are available, effectively hallucinating tool arguments. Disabling parallel tool calls can mitigate this behavior.