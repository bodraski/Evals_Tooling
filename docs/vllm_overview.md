# vLLM для evals reasoning-моделей: короткая заметка

## 1. Зачем vLLM

**vLLM** — это inference/server layer: он запускает модель и отдаёт ответы через OpenAI-compatible API. В нашем контексте он находится между `Inspect` и моделью:

```text
Inspect eval → vLLM OpenAI-compatible API → модель → vLLM response → Inspect normalization → scorer/solver
```

Главный риск для reasoning-моделей: ответ может разделяться на два поля:

```text
content / final text          = финальный ответ
reasoning_content / reasoning = reasoning-блок
```

Если `content` пустой, а `reasoning_content` содержит `ANSWER: ...`, надо понять, где проблема:

```text
модель не дала final answer
или vLLM parser неправильно разделил reasoning/final
или Inspect normalization потеряла поле
```

## 2. Основные части конфигурации vLLM

### 2.1. Model id

Пример:

```bash
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B
```

Для баг-репорта обязательно фиксировать точный `model id`.

### 2.2. OpenAI-compatible server

vLLM можно запускать как HTTP server, совместимый с OpenAI API:

```bash
vllm serve MODEL_NAME --host 0.0.0.0 --port 8000
```

К нему потом можно обращаться через OpenAI client с `base_url=http://localhost:8000/v1`.

### 2.3. Reasoning parser

Для reasoning-моделей это ключевой параметр:

```bash
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B \
  --reasoning-parser deepseek_r1
```

Для Qwen3 через Inspect пример выглядит так:

```bash
inspect eval math.py --model vllm/Qwen/Qwen3-8B -M reasoning_parser=qwen3
```

Если parser не указан или выбран неверно, vLLM может неправильно разделить reasoning и final answer.

### 2.4. Chat template

Для chat-моделей vLLM использует chat template из tokenizer config или заданный вручную template. 
Неверный template может привести, кроме прочего, к тому, что модель не пишет финальный ответ в ожидаемое место.

### 2.5. Token budget

Фиксировать нужно оба уровня:

```text
--max-model-len   # context limit на стороне vLLM
max_tokens        # output budget в конкретном запросе/eval
```

Для reasoning-моделей это особенно важно: модель может потратить budget на reasoning и не успеть написать финальный `content`.

### 2.6. Throughput / память / parallelism

Для воспроизводимости желательно фиксировать:

```text
vLLM version
GPU type, если есть
--gpu-memory-utilization
--tensor-parallel-size
--pipeline-parallel-size
--max-num-seqs
--max-num-batched-tokens
```

Даже если мы сами не запускаем GPU, эти поля нужны для нормального bug report.

## 3. Что проверять в логах

Минимальная таблица для каждого подозрительного sample:

```text
log file
model id
provider path: vLLM / API provider / Together / OpenRouter / official API
vLLM command / config
reasoning_parser
chat_template
max_model_len
max_tokens
raw content
raw reasoning_content
Inspect .completion
Inspect ContentReasoning
solver/scorer
score
```

## 4. References: куда смотреть

### vLLM docs

- vLLM Quickstart / OpenAI API server. Link: https://docs.vllm.ai/en/stable/getting_started/quickstart/#openai-compatible-server

- vLLM OpenAI-Compatible Server  `Chat Template` section.
  Link: https://docs.vllm.ai/en/stable/serving/openai_compatible_server/#chat-template

- vLLM Reasoning Outputs  
  Link: https://docs.vllm.ai/en/latest/features/reasoning_outputs/

- vLLM `serve` CLI args  
  Link: https://docs.vllm.ai/en/stable/cli/serve/

### Inspect docs

- Inspect Model Providers / vLLM  
  Link: https://inspect.aisi.org.uk/providers.html 
  Смотреть: `vLLM`, `Tool Use and Reasoning`, note about passing model-specific parser via `-M`.

- Inspect Reasoning  
  Link: https://inspect.aisi.org.uk/reasoning.html  
  Смотреть: `vLLM / SGLang`; example `-M reasoning_parser=qwen3`.

- Inspect Eval Logs  
  Link: https://inspect.aisi.org.uk/eval-logs.html  
  Смотреть: `Overview`; where logs are written and how they are structured.

- Inspect Log Viewer  
  Link: https://inspect.aisi.org.uk/log-viewer.html  
  Смотреть: `Overview`; message histories, scoring decisions, metadata.
