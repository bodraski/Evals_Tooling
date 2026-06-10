## Tool calling flags for vLLM

When serving with vLLM, tool calling requires two additional flags:

```bash
--enable-auto-tool-choice
--tool-call-parser <parser name>
```

`--enable-auto-tool-choice` flag is a mandatory Auto tool choice. It tells vLLM that you want to enable the model to generate its own tool calls when it deems appropriate.

`--tool-call-parser <parser>` choose the parser based on your model family (see vLLM tool calling docs).

- Both flags are required together. --enable-auto-tool-choice tells vLLM to allow the model to decide when to invoke tools, while --tool-call-parser <parser> provides the actual parsing logic that extracts structured tool_calls from the model's raw text output (the specific format varies by model family and chat template). 
- Without the parser, vLLM has no way to interpret the model's output! And vllm won't even start outputting the TypeError: Error: --enable-auto-tool-choice requires --tool-call-parser

## vLLM Tool Calling: Launch Scenarios

| Scenario | vLLM starts? | Eval result | Error |
|----------|-------------|-------------|-------|
| Neither flag set | ✅ Yes | ❌ Fails at runtime | `400 BadRequestError: "auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set` |
| `--enable-auto-tool-choice` only | ❌ No | ❌ Never reached | `TypeError: Error: --enable-auto-tool-choice requires --tool-call-parser` (startup crash) |
| Both flags set correctly | ✅ Yes | ✅ Works | — |

### Why these flags can't be set automatically by Inspect or the client

The flags are **vLLM server startup arguments** — they must be passed when launching the container/process, before any client connects. Neither Inspect AI nor any API client can set them retroactively on a running server.

Additionally, `--tool-call-parser` is **model-specific**: the correct value depends on the model being served (`hermes`, `mistral`, `deepseek_r1`, `llama3_json`, etc.). There is no universal default that works across models, so auto-detection would require either a registry of model→parser mappings or a convention that doesn't currently exist in vLLM.

### Why not fall back to `emulate_tools`?
 
Inspect AI's `emulate_tools=True` is a client-side workaround that converts tool calls into plain-text prompts, bypassing native tool calling entirely. It was considered as a fallback but rejected for two reasons:
 
- **Permanent mutation**: setting `emulate_tools` on a model object affects all subsequent tasks in the same eval run, even unintended ones — the side effect is silent and hard to trace.
- **Result validity**: emulated tool calling produces different model behaviour than native tool calling, making results incomparable to real-world usage.
The correct approach is to fail explicitly with an actionable error when the server flags are missing, rather than silently degrading to emulation.
 
**Correct launch example:**
```bash
docker run -d --gpus all -p 8000:8000 vllm/vllm-openai:latest \
  --model <model-name> \
  --enable-auto-tool-choice \
  --tool-call-parser <parser>   # e.g. hermes, mistral, deepseek_r1
```
---

## How to Choose the Right `--tool-call-parser` in vLLM

### Step 1: Check the model card
Look for "tool calling" or "function calling" section on HuggingFace.
The recommended parser is usually stated explicitly.

### Step 2: Check vLLM docs
Search your model family at:
`https://docs.vllm.ai/en/latest/features/tool_calling/`

### Step 3: Inspect the chat template
Open `tokenizer_config.json` → `chat_template` field and look for the tool call format:

| Pattern in template | Parser to use |
|---|---|
| `<tool_call>...</tool_call>` | `hermes` |
| `<function_calls>` / `<｜tool▁sep｜>` (DSML) | `deepseek_v3` / `deepseek_v4` |
| XML tags (`<tools>`, `<function_calls>`) | `qwen3_xml` |
| Python list syntax (`[func(...)]`) | `pythonic` |
| `[TOOL_CALLS]` token | `mistral` |

### Step 4: For fine-tuned / distilled models
Default to the base model's parser, but verify with a smoke test,
since fine-tuning sometimes changes the tool call format.

### Step 5: Check GitHub issues for your model
Known parser bugs (e.g. `qwen3_coder` infinite stream on long inputs)
are often reported before being fixed. Search:
`site:github.com/vllm-project/vllm <model-name> tool`

---

## Smoke Test

Start vLLM with `--enable-auto-tool-choice --tool-call-parser <parser>`, then:

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get weather for a city",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"]
        }
    }
}]

response = client.chat.completions.create(
    model="your-model-name",
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
    tools=tools,
    tool_choice="required",  # force a tool call
    max_tokens=256
)

choice = response.choices[0]
print("finish_reason:", choice.finish_reason)   # expect: "tool_calls"
print("tool_calls:   ", choice.message.tool_calls)  # expect: non-empty list
print("content:      ", choice.message.content)     # expect: None or empty
```

**Parser is working correctly if:**
- `finish_reason` is `"tool_calls"`
- `tool_calls` contains a valid call with `name` and `arguments`
- `content` is `None` or empty

**Red flags:**
- `finish_reason: "stop"` with text response → parser mismatch
- `tool_calls` is `None` or empty → wrong parser or missing `--enable-auto-tool-choice`
- Infinite stream / repeated tokens → known parser bug, try an alternative

---

Example of container start-up:
---

### Types of parsers

| Parser | Model / model family |
|---|---|
| `hermes` | Nous Research Hermes 2 Pro+, Qwen2.5, Qwen3 and other Hermes-compatible |
| `mistral` | Mistral 7B Instruct v0.3+, other Mistral function-calling models |
| `llama3_json` | Llama 3.1, 3.2 (JSON-format) |
| `llama4_json` | Llama 4 (JSON-format, alternative pythonic) |
| `llama4_pythonic` | Llama 4 (recommended) |
| `pythonic` | all models using Python-list format for tool calls |
| `granite` | IBM Granite 3.x Instruct |
| `granite-20b-fc` | IBM Granite 20B Function Calling |
| `internlm` | InternLM2 (ubstable for internlm2-chat-7b) |
| `jamba` | AI21 Jamba-1.5 |
| `deepseek_v3` | DeepSeek-V3, DeepSeek-R1 and compatible |
| `deepseek_v4` | DeepSeek-V4-Pro, DeepSeek-V4-Flash |
| `phi4_mini_json` | Microsoft Phi-4-mini |
| `xlam` | Salesforce xLAM (JSON-arrays, supports `<think>` tags) |
| `minimax` | MiniMax models |
| `openai` | OpenAI OSS models |
| `qwen3_xml` | Qwen3-Coder (XML-format) |
| `kimi_k2` | moonshotai/Kimi-K2-Instruct |
| `glm47` | zai-org/GLM-4.7, GLM-4.7-Flash |
| `glm45` | zai-org/GLM-4.5, GLM-4.5-Air, GLM-4.6 |
| `longcat` | meituan-longcat/LongCat-Flash-Chat |
| `hunyuan_a13b` | Hunyuan-A13B (+ `--reasoning-parser hunyuan_a13b`) |
| `cohere_command3` | CohereLabs/command-a-reasoning-08-2025 (requires `cohere_melody`) |

> **NB**
>
> - `pythonic` — parser for models, calling tool calls such as Python-list instead of JSON;
>   inherently supports parallel tool calls and removes ambiguity around the JSON schema
> - for Llama 4 pythonic-format is recommended
> - `xlam` supports several output formats: JSON-arrays and JSON inside `<think>` tags.
> - custom parser can be registered via `--tool-parser-plugin <path>`,
>   and then it's called using `--tool-call-parser`.

```bash
    --enable-auto-tool-choice \
    --tool-parser-plugin <absolute path of the plugin file>
    --tool-call-parser example \
    --chat-template <your chat template> \
```

