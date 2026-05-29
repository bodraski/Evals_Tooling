# Findings: reasoning tag handling in vLLM + Inspect AI

## Setup

- Model: `casperhansen/deepseek-r1-distill-qwen-7b-awq`
- vLLM `0.21.0`, Docker, GPU instance
- Inspect AI connected via `model_base_url`
- Logging via `mitmweb` proxy between Inspect AI and vLLM

---

## Finding 1: where `<think>` tags are parsed

Without `--reasoning-parser` on vLLM, the raw response has `<think>` tags in `content`:

```json
"content": "<think>...reasoning...</think>\nfinal answer",
"reasoning": null
```

Inspect AI's fallback parser strips the tags and produces:

```json
{"type": "reasoning", "internal": "think", ...}
{"type": "text", "text": "final answer"}
"completion": "final answer"
```

With `--reasoning-parser deepseek_r1`, vLLM separates them at the server level:

```json
"content": "final answer",
"reasoning": "...reasoning..."
```


Side effect: without `--reasoning-parser`, reasoning tokens are counted as `completion_tokens`, inflating the metric.

---

## Finding 2: fallback parser breaks on nested `<think>` tags

The fallback parser uses a greedy first-match strategy:

- finds `<think>` → starts capturing reasoning  
- finds first `</think>` → stops, everything after goes to `text`

No nesting awareness. If the model's reasoning contains a literal `<think>hello</think>` as part of its output (e.g. when reasoning about the tag itself), the parser splits at the inner `</think>`, truncating the reasoning block mid-sentence.

**Reproduced with:** a dataset sample asking the model to construct the string `<think>hello</think>` by describing its parts. The model correctly reasoned through it but the fallback parser cut the reasoning at the inner closing tag.

Result: reasoning truncated, `text` block starts mid-sentence, `completion` passed to scorer is incomplete. No error raised.

**USED DATASET & TASK**

```python
@task
def make_it_think():
    return Task(
        dataset = [
            
            Sample(
                input="Print exactly this text and nothing else: &lt;think&gt;hello&lt;/think&gt;",
                target="<think>"  
            ),
            Sample(
                input="Print a less-than sign, then the word think, then a greater-than sign, then hello, then less-than, slash, think, greater-than",
                target="<think>"
            ),
            Sample(
                input="Print exactly the content of this variable: s = '<' + 'think' + '>hello</' + 'think' + '>'",
                target="<think>" 
            ),
            Sample(
                input="Print exactly the content of this variable: s = '<' plus 'think' plus '>hello</' plus'think' plus'>', where plus means concatenation",
                target="<think>" 
            ),
        ],
        solver=[
            chain_of_thought(),
            generate(),
        ],
        scorer=includes()
    )
```



---

## Finding 3: `<think>` in user prompt causes tokenization failure

`<think>` is a special token (id `151648`) in the DeepSeek-R1 tokenizer vocabulary. When it appears in a user message, it is tokenized as a single special token — not as the literal character sequence `<`, `t`, `h`, `i`, `n`, `k`, `>`.

Verified via `prompt_logprobs`:

```json
{"151648": {"decoded_token": "<think>"}}
```

Entire prompt `count <think> tags` = 7 tokens total.

**Consequence:** if a dataset contains `<think>` in prompts and a reasoning model is used, the model receives a special token instead of text. It does not understand the question as intended. The eval completes without errors, scores are meaningless.

Workaround: describe the tag without using the literal characters, e.g.:

```
"Print a less-than sign, then the word think, then a greater-than sign..."
```

This forces character-by-character tokenization. The model can then reason about and produce `<think>hello</think>` correctly.

**Note:**  ⚠️  this is a vLLM/tokenizer-level issue, not fixable from Inspect AI side. Should be documented as a known limitation when using reasoning models with datasets that reference `<think>` tags.

---

## Tools used

`mitmweb` as a transparent proxy for raw request/response inspection:

```bash
mitmweb --mode reverse:http://localhost:8765 --listen-port 8766
# web UI: http://localhost:8081
```

Session-aware JSON logging via mitmproxy addon with `/proxy/new_session` endpoint for log rotation between eval runs.