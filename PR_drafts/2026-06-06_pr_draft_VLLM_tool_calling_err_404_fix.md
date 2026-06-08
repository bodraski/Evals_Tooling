**Suggested title:** Bug: vLLM provider raises opaque `NotFoundError` (404) when tool calling is not configured; no actionable guidance shown.

---

## Summary

When a vLLM server is started without `--enable-auto-tool-choice` and `--tool-call-parser`, any eval that uses tools fails with a raw `NotFoundError: 404` from the OpenAI-compatible client. The error gives no indication of what is wrong or how to fix it.

## Environment

- **inspect-ai**: 0.3.222
- **vLLM**: 0.21.0

## Root Cause

`VLLMAPI.generate()` delegates to `OpenAICompatibleAPI.generate()`, which only catches 3 types of errors:
 - `BadRequestError`
 - `UnprocessableEntityError`
 - `PermissionDeniedError`
  A 404 from a server not configured for tool calling surfaces as an unhandled `NotFoundError` with no context.

## Proposed fix

### Option 1

Catch `NotFoundError` in `VLLMAPI.generate()` and raise a `RuntimeError` with an actionable message when tools were present in the request:


```python
except NotFoundError as ex:
    if len(tools) > 0:
        raise RuntimeError(
            "vLLM returned 404 for a request with tools. "
            "Either start vLLM with --enable-auto-tool-choice and "
            "--tool-call-parser=<parser> (e.g. hermes, llama3_json, mistral), "
            "or use -M emulate_tools=true as a fallback."
        ) from ex
    raise
```

### Option 2

Automatically fall back to `emulate_tools` with a warning. 
Better UX out of the box, but silently uses a fallback tool calling method which is less accurate.

```python
except NotFoundError:
            if len(tools) > 0:
                warn_once(
                    logger,
                    "vLLM returned 404 for a request with tools — "
                    "server is not configured for native tool calling. "
                    "Falling back to tool emulation. For better results, "
                    "restart vLLM with --enable-auto-tool-choice and "
                    "--tool-call-parser=<parser> (e.g. hermes, llama3_json, mistral).",
                )
                self.emulate_tools = True
                return await super().generate(input, tools, tool_choice, config)
            raise
```


Happy to proceed with whichever approach you prefer.


