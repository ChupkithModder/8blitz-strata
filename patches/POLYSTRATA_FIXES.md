# PolyStrata Server & Parser Fixes for Inkling 2 Mini

If your PolyStrata checkout (`terminal/serve/frontend.py` and `terminal/serve/server.py`) does not already include these fixes, apply them before running Inkling with OpenCode or Oh My Pi (`omp`):

## 1. `terminal/serve/frontend.py` — `InklingOutputParser` Tool-Call Recovery
- **`_ink_call_end(s, final)`:** Include `INK_STOP` (`<|content_model_end_sampling|>`) alongside `INK_END` and `INK_MSG` so a stop marker emitted as text tokens still terminates a tool call cleanly.
- **`_call(self, body)`:**
  - Strip optional markdown fences (`_JSON_FENCE` + trailing ```` ``` ````).
  - Fall back to `lenient_json(text)` when strict `json.loads(text)` raises `ValueError` (handles trailing commas, single quotes, Python `True`/`False`/`None`, unescaped newlines).
  - Accept `"args"`, `"arguments"`, `"parameters"`, `"input"`, or direct unwrapped argument dicts when `"name"` is omitted and `self.header.strip()` is a declared tool.
  - Decode stringified `args` (`if isinstance(args, str): json.loads / lenient_json`).
  - Increment `self.unread` / `self.unread_names` on unparseable calls so `/metrics` and logs surface them.
- **`_read()` (`header` mode):** Allow leading whitespace before a tool name (e.g. `<|message_model|>\nquestion<|content_invoke_tool_json|>`) by checking whitespace starting at `lead = len(s) - len(s.lstrip())`, while still falling back to `"content"` mode if whitespace appears *inside* non-tag text.

## 2. `terminal/serve/server.py` — Reasoning Retry & CUDA Crash Recovery
- **`reasoning_close_retry`:** When `self.call_format == "inkling"`, inject `INK_END + INK_MSG` (`<|end_message|><|message_model|>`) instead of `</think>`.
- **CUDA `illegal memory access` auto-respawn:** In the engine stdout pump loop (`if line.startswith(FATAL_PREFIXES)`), also match `(line.startswith("ERR ") and "illegal memory access" in line.lower())` so a poisoned CUDA context is terminated immediately and the next request auto-respawns a clean engine instead of repeating `reset: an illegal memory access was encountered`.
- **Numeric `reasoning_effort` clamping:** In `inkling_template_kwargs()`, clamp numeric `reasoning_effort` into `[0.0, 0.99]` so clients passing `1.0` do not trigger a Jinja template error.
