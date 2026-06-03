# PR draft: External vLLM endpoint smoke check

## Summary

Add a lightweight smoke check for already-running external vLLM endpoints used via `VLLM_BASE_URL`.

This is a setup/debugging helper, not a benchmark and not a full vLLM validator. Its purpose is to catch cheap endpoint mistakes and verify that the endpoint can complete one minimal `/v1/chat/completions` request before a user launches a larger Inspect eval.

The smoke check should:

1. Require `--base-url` and `--model`.
2. Optionally collect `/version` for debugging context, if exposed.
3. Try `/v1/models` to check auth and requested model availability, if exposed.
4. Run one tiny chat-completions probe by default.
5. If `/v1/models` is unavailable or hidden by a proxy, report that but still allow the chat probe to proceed unless strict mode is requested.

The check does **not** validate chat templates, reasoning parsers, tool-call parsers, or model quality. It only checks externally observable endpoint behavior.

---

## Motivation

Users can point Inspect to an existing vLLM server via `VLLM_BASE_URL`.

Before running a large eval, users may want to know:

- is this endpoint reachable?
- is my API key accepted?
- is the model id I plan to use accepted by the endpoint?
- does `/v1/chat/completions` work for this model?
- does the minimal response have a non-empty visible content shape?

Today users can manually do this with curl. The smoke check would package that first debugging step into a repeatable command/report.

---

## Why this adds value beyond normal log analysis

This feature is not meant to replace log analysis. It prevents users from reaching log analysis for trivial setup failures.

### Without a smoke check

A user may start a costly eval and only later discover:

- wrong host or port;
- missing API key;
- wrong API key;
- model id mismatch;
- `/v1/models` works, but `/v1/chat/completions` fails;
- `/v1/models` is hidden by a gateway, so the user cannot rely on model introspection;
- the first model call fails with a basic HTTP or model-id error.

They then interrupt, inspect logs, and manually reconstruct the endpoint state.

### With a smoke check

The user can run one command and get an immediate report:

```text
External vLLM smoke check

[SKIPPED] version
  /version not exposed

[WARN] models
  /v1/models unavailable or hidden by proxy; continuing to chat probe

[PASS] chat_probe
  finish_reason: stop
  content_empty: false
```

or:

```text
[FAIL] chat_probe
  /v1/chat/completions returned 401 Unauthorized
```

The main value is the chat probe. `/version` and `/v1/models` are useful when available, but should not be treated as mandatory for every external deployment.

---

## Existing coverage and distinction

Inspect already has vLLM provider tests. Existing tests cover provider-level behavior such as:

- local vLLM chat generation returns a non-empty completion;
- the `vllm-completions` provider works with `/v1/completions`;
- logprobs, echo mode, usage, and extra-body passthrough for completions;
- LoRA/shared-server plumbing.

This proposal is different:

- it targets **user-managed external vLLM endpoints**;
- it is user-facing setup tooling, not only a provider test;
- it does not start vLLM;
- it does not require GPU access;
- it checks the actual endpoint the user intends to run evals against.

---

## Why not just strengthen the existing vLLM provider test?

The existing vLLM provider test already sends a tiny prompt and checks that local vLLM generation returns a non-empty completion.

That test could be strengthened, for example by also checking:

- `response.stop_reason is not None`;
- `response.message.text.strip()` is non-empty;
- `response.completion` is consistent with the assistant message.

That would be a reasonable small provider-test improvement.

However, it would still be a developer/provider test for local vLLM. It would not help a user who has an already-running external vLLM endpoint and wants to check whether their actual `VLLM_BASE_URL`, API key, and model id are usable before launching an eval.

So this proposal is about a user-facing external endpoint smoke check.

---

## Relationship to related PRs / branches

This proposal should stay clearly separate from reasoning parsing and reasoning parser autodetection work.

### Reasoning parser autodetection

A related branch works on detecting known tag-based reasoning model families, configuring local vLLM/SGLang reasoning parsers where possible, and avoiding unsafe `<think>` fallback parsing for unknown models.

That is a different layer.

- Reasoning-parser work decides how Inspect should parse or configure reasoning for known model families.
- This smoke check does not choose a reasoning parser.
- This smoke check does not maintain a model-family mapping.
- This smoke check does not decide whether `<think>` tags should be parsed.
- This smoke check only observes whether an external endpoint can complete one minimal chat request.

### Nested `<think>` parsing

A related branch works on parsing nested `<think>` blocks correctly.

That is also a different layer.

- Nested-`<think>` work modifies parser behavior.
- This smoke check does not parse or normalize `<think>` content.
- If the chat probe observes raw reasoning-looking markup, it may report the response shape, but it should not fix or reinterpret it.

### Token truncation warnings

There is also related work around warning when model output is truncated by token or model length. That concerns runtime eval diagnostics and scoring/logging behavior.

This smoke check is narrower:

- it is run before or outside a larger eval;
- it sends at most one tiny chat request;
- it does not add passive diagnostics to every eval response;
- it does not change scoring behavior.

If the chat probe returns empty content with a length stop, the smoke check can report that observed shape. It should not replace runtime truncation warnings.

---

## Non-goals

This proposal should **not**:

- infer or validate exact `--chat-template`;
- infer or validate exact `--reasoning-parser`;
- infer or validate exact `--tool-call-parser`;
- duplicate reasoning-parser autodetection work;
- parse or fix `<think>` content;
- test tool calling by default;
- run multi-turn probes;
- run token sweeps;
- run SWE-bench or any eval task;
- compare model quality;
- automatically run before every eval unless explicitly requested.

This check should be framed as endpoint sanity, not semantic certification.

---

## Why not inspect launch configuration?

For external user-managed vLLM servers, Inspect generally only has:

- base URL;
- API key;
- requested model id.

It usually cannot:

- SSH into the host;
- inspect Docker;
- read the vLLM launch command;
- read the server-side chat template;
- read `--reasoning-parser`;
- read `--tool-call-parser`.

The OpenAI-compatible API may expose model ids and sometimes `/version`, but it should not be assumed to expose full launch configuration.

Therefore the smoke check should not claim to validate vLLM configuration. It only checks externally observable behavior.

---

## Why not rely on `/v1/models`?

For a standard vLLM OpenAI-compatible server, `/v1/models` is expected to exist and is useful. It can catch:

- wrong API key;
- wrong endpoint;
- requested model id not served;
- model aliases that differ from the Hugging Face model path.

However, in real external deployments, `/v1/models` may be unavailable or unreliable because:

- a proxy/gateway hides it;
- a hosted wrapper exposes only selected routes;
- the endpoint is OpenAI-compatible but not a vanilla vLLM server;
- auth policies differ per route;
- the server exposes a custom served model name.

Therefore `/v1/models` should be used opportunistically:

- if available, report served model ids and whether the requested model is present;
- if unavailable, report `warn` or `skipped`;
- do not block the chat probe unless strict mode is enabled.

The chat probe is the more meaningful smoke-test step because it exercises the actual route Inspect needs for chat evals.

---

## Why not use reasoning-model lists?

Reasoning-parser/autodetection work may maintain lists of known model families and parser mappings. This smoke check should not duplicate that logic.

Reasons:

1. External vLLM servers may expose aliases rather than original Hugging Face model names.
2. `/v1/models` usually does not expose parser configuration.
3. Parser choice is a separate layer.
4. The smoke check should remain operational and simple.
5. Inferring parser correctness from a model id would overclaim.

The smoke check may report the served model id. It should not decide which reasoning parser should have been used.

---

## Proposed user experience

Possible command:

```bash
inspect vllm-smoke \
  --base-url "$VLLM_BASE_URL" \
  --api-key "$VLLM_API_KEY" \
  --model "MODEL_ID"
```

Alternative naming:

```bash
inspect doctor vllm ...
inspect check-vllm ...
```

The exact command name is open. If maintainers dislike adding a CLI command immediately, the first version could be an experimental helper script or documentation recipe.

---

## Default behavior

Default behavior should include one chat probe.

Reason: if the feature is called a smoke test, it should actually exercise the chat-completions path.

Default steps:

1. Try `GET /version`.
   - If available, report version.
   - If unavailable, skip.
2. Try `GET /v1/models`.
   - If available, report model ids and requested-model presence.
   - If unavailable, warn/skip and continue.
3. Send one tiny `/v1/chat/completions` request.
   - This is the core smoke test.
   - It should be clearly documented that one generation request will be sent.

Escape hatch:

```bash
--no-chat-probe
```

for users who want endpoint-only checks.

Strict mode:

```bash
--strict
```

could make `/v1/models` absence or requested-model absence a failure. Without strict mode, `/v1/models` absence should not block the chat probe.

---

## Chat probe

### Probe prompt

```text
Final answer only. Do not explain. Reply exactly: VLLM_OK_7G2
```

Suggested request:

```json
{
  "model": "MODEL_ID",
  "messages": [
    {
      "role": "user",
      "content": "Final answer only. Do not explain. Reply exactly: VLLM_OK_7G2"
    }
  ],
  "temperature": 0,
  "max_tokens": 64
}
```

`max_tokens` should be configurable:

```bash
--max-tokens 64
```

For reasoning models, users may choose a slightly larger value such as 128. But the default should stay small.

### What to report

Report:

- HTTP success/failure;
- `finish_reason`;
- content empty/non-empty;
- exact sentinel match true/false;
- usage present/absent;
- latency;
- whether `tool_calls` field is present;
- whether a structured reasoning field is present.

Do not treat exact-match failure as a hard failure. It is weak evidence only.

Hard failure only for:

- HTTP error;
- auth error;
- malformed response;
- chat endpoint unavailable;
- model rejected by chat-completions endpoint.

Example successful report:

```json
{
  "name": "chat_probe",
  "status": "pass",
  "evidence": {
    "finish_reason": "stop",
    "content_empty": false,
    "exact_match": true,
    "usage_present": true,
    "latency_seconds": 1.2
  }
}
```

Example non-exact but non-empty response:

```json
{
  "name": "chat_probe",
  "status": "warn",
  "message": "Chat endpoint returned content, but it did not exactly match the sentinel. This may be model behavior, not endpoint failure.",
  "evidence": {
    "content_empty": false,
    "exact_match": false
  }
}
```

Example empty response:

```json
{
  "name": "chat_probe",
  "status": "warn",
  "message": "Chat endpoint returned no visible content. Inspect the response shape and generation settings.",
  "evidence": {
    "finish_reason": "length",
    "content_empty": true,
    "reasoning_field_present": true
  }
}
```

This should remain descriptive. It should not conclude that the server is misconfigured.

---

## `/version` check

Request:

```http
GET /version
```

Possible statuses:

- `pass`: endpoint returns version information;
- `skipped`: endpoint does not expose `/version`;
- `warn`: endpoint returns unexpected response.

This is useful for debugging context but is not essential.

---

## `/v1/models` check

Request:

```http
GET /v1/models
```

Possible statuses:

- `pass`: model list returned and requested model is present;
- `warn`: model list returned but requested model is absent;
- `warn`: endpoint unavailable, hidden, or response shape unexpected;
- `fail`: unauthorized or connection failure, if this prevents the check entirely.

In non-strict mode, absence of `/v1/models` should not block the chat probe.

In strict mode, requested model absence can be a hard failure.

Require `--model`; do not silently choose the first model from the list.

If `/v1/models` is unavailable:

```json
{
  "name": "models",
  "status": "warn",
  "message": "/v1/models is unavailable or hidden; continuing to chat probe because --model was provided.",
  "evidence": {
    "http_status": 404
  }
}
```

---

## Report schema

```python
class VLLMSmokeCheck(BaseModel):
    name: str
    status: Literal["pass", "warn", "fail", "skipped"]
    message: str | None = None
    evidence: dict[str, Any] = Field(default_factory=dict)


class VLLMSmokeReport(BaseModel):
    base_url: str
    model: str
    chat_probe: bool
    strict: bool
    checks: list[VLLMSmokeCheck]
```

Human-readable output should be concise.

Example:

```text
External vLLM smoke check

base_url: http://localhost:8765
model: Qwen/Qwen3-8B
chat_probe: true
strict: false

[SKIPPED] version
  /version not exposed

[WARN] models
  /v1/models unavailable or hidden; continuing to chat probe

[PASS] chat_probe
  finish_reason: stop
  content_empty: false
  exact_match: true
```

---

## Implementation sketch

Potential helper location:

```text
src/inspect_ai/model/_providers/vllm_smoke.py
```

or:

```text
src/inspect_ai/model/_vllm_smoke.py
```

Pseudo-code:

```python
async def smoke_check_vllm_endpoint(
    base_url: str,
    api_key: str | None,
    model: str,
    *,
    chat_probe: bool = True,
    strict: bool = False,
    max_tokens: int = 64,
    timeout: float = 20,
) -> VLLMSmokeReport:
    root_url, api_url = normalize_vllm_base_url(base_url)

    checks = []

    checks.append(await check_version(root_url, api_key, timeout))

    models_check = await check_models(
        api_url,
        api_key,
        model,
        timeout,
        strict=strict,
    )
    checks.append(models_check)

    if chat_probe:
        if strict and models_check.status == "fail":
            checks.append(
                VLLMSmokeCheck(
                    name="chat_probe",
                    status="skipped",
                    message="Skipping chat probe because strict model check failed.",
                )
            )
        else:
            checks.append(
                await check_chat_probe(
                    api_url,
                    api_key,
                    model,
                    max_tokens=max_tokens,
                    timeout=timeout,
                )
            )
    else:
        checks.append(
            VLLMSmokeCheck(
                name="chat_probe",
                status="skipped",
                message="Chat probe disabled.",
            )
        )

    return VLLMSmokeReport(
        base_url=base_url,
        model=model,
        chat_probe=chat_probe,
        strict=strict,
        checks=checks,
    )
```

---

## Tests

Unit tests should mock HTTP responses and should not require vLLM/GPU.

Candidate file:

```text
tests/model/providers/test_vllm_smoke.py
```

1. Version pass.
2. Version skipped on 404.
3. Models unauthorized.
4. Requested model present.
5. Requested model absent, non-strict.
6. Requested model absent, strict.
7. `/v1/models` unavailable, non-strict.
8. Model omitted.
9. Chat probe normal response.
10. Chat probe non-exact but non-empty.
11. Chat probe HTTP failure.
12. Chat probe disabled.

---

## Latency and token budget

Default mode:

- one optional `/version` request;
- one `/v1/models` request if available;
- one chat-completion request;
- default `max_tokens=64`;
- configurable `--max-tokens`;
- short timeout;
- no retries unless aligned with existing client conventions.

The command should print clearly that it is sending one generation request.

For no-generation mode:

```bash
--no-chat-probe
```

---

## Examples where this helps

### Example 1: wrong API key

```text
[FAIL] chat_probe
  HTTP 401 Unauthorized
  Suggested action: check API key / Authorization header.
```

or, if `/v1/models` catches it first:

```text
[FAIL] models
  HTTP 401 Unauthorized
```

### Example 2: `/v1/models` hidden, but chat works

```text
[WARN] models
  /v1/models unavailable or hidden; continuing to chat probe

[PASS] chat_probe
  content_empty: false
```

This confirms that model-list introspection is unavailable, but the actual chat path works.

### Example 3: wrong model id

```text
[WARN] models
  requested model Qwen/Qwen3-8B not listed; served models include qwen3

[FAIL] chat_probe
  model not found
```

This points directly to model alias mismatch.

### Example 4: `/v1/models` works but chat completions fail

```text
[PASS] models
  requested_model_present: true

[FAIL] chat_probe
  /v1/chat/completions returned 404
```

This distinguishes “the server lists models” from “the chat-completions route works.”

### Example 5: chat path works but response shape is odd

```text
[WARN] chat_probe
  content_empty: true
  finish_reason: length
  reasoning_field_present: true
```

This does not prove misconfiguration. It tells the user that the endpoint can be reached, but the minimal chat response shape deserves inspection before running a large eval.

---

## Open questions

1. Should the command be named `vllm-smoke`, `check-vllm`, or `doctor vllm`?
2. Should chat probe be default-on?
   - Recommendation: yes if the feature is called a smoke test.
   - Provide `--no-chat-probe` for endpoint-only checks.
3. Should `/v1/models` absence be a failure?
   - Recommendation: no in non-strict mode.
   - It should be a warning/skipped check, because external deployments may hide it.
4. Should missing requested model be a failure?
   - Recommendation: warn in non-strict mode, fail in strict mode.
   - Let the chat probe be the decisive test of whether the endpoint accepts the requested model.
5. Should `/version` be included?
   - Optional debugging context only.
   - Never required.
6. Should the first implementation use existing OpenAI-compatible client utilities?
   - Recommendation: yes, to match repo conventions.
7. Should this first land as docs/an example script rather than core CLI?
   - Reasonable alternative if maintainers are unsure about product surface.

---

## Minimal PR

```text
- Add external vLLM smoke-check helper/command.
- Require base_url and model.
- Try /version if available; skip if missing.
- Try /v1/models; warn/skip if unavailable in non-strict mode.
- Run one tiny chat probe by default.
- Add --no-chat-probe.
- Add --strict.
- Add mocked tests.
```

No parser changes.  
No model-family reasoning logic.  
No tool-calling validation.  
No multi-turn validation.  
No eval integration by default.
