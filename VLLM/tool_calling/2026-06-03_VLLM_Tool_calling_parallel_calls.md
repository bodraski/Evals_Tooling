## Parallel Tool Calls

### The Problem

Many models support parallel tool calls by default, emitting multiple
tool calls in a single assistant turn. For evaluations involving
**dependent (sequential) tool calls**, this can silently corrupt results:
the model generates downstream calls before upstream results are
available, effectively hallucinating arguments.

This is a model-side generation behavior, distinct from how Inspect
executes tool calls internally (controlled by `@tool(parallel=True/False)`).

### Controlling Generation via Inspect

To disable parallel tool call generation, set `parallel_tool_calls=False`
in `GenerateConfig`:

```python
from inspect_ai.model import GenerateConfig

eval(
    task,
    model="openai/your-model",
    generate_config=GenerateConfig(parallel_tool_calls=False)
)
```

This passes `parallel_tool_calls: false` in the API request. vLLM
forwards it to the model via the chat template.

> **Note:** Smaller and quantized models may ignore this flag and still
> emit parallel calls. This is a model limitation, not a configuration
> bug — treat it as a valid eval result rather than correcting for it.

### When to Disable Parallel Tool Calls

| Benchmark type | Recommendation |
|---|---|
| Sequential-only (e.g. BFCL nested / multi-turn) | Set `parallel_tool_calls=False`, document it explicitly |
| Parallel-only | Leave default |
| Mixed (sequential + parallel) | Leave default — ability to distinguish is what you're measuring |

### A Note on Quantized Models

Failure to respect call ordering has been observed primarily in quantized
models. If sequential tool call compliance is a metric in your eval,
report `parallel_tool_calls` setting alongside model precision as part
of your experimental conditions.

---
> **Note on documentation:** The Inspect reference currently marks
> `parallel_tool_calls` as "OpenAI and Groq only". In practice, this
> parameter is passed as a standard OpenAI-compatible API field and
> works with any provider that supports it, including vLLM.
> Verify behavior with a smoke test for your specific model and vLLM version.