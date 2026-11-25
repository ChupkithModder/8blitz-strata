# Agent Setup Runbook (`inkling2-polystrata-setup`)

This document is written so an AI coding agent (OpenCode, Oh My Pi, Claude Code, Codex, Antigravity, etc.) can automatically install, configure, tune, and verify **Inkling 2 Mini** on **PolyStrata** for the user's OS and hardware.

---

## Step 1: Detect Hardware & OS

1. **OS:** Windows, Linux, or macOS.
2. **GPU & VRAM:**
   - NVIDIA: `nvidia-smi --query-gpu=name,memory.total --format=csv,noheader`
   - AMD (Linux): `rocm-smi --showmeminfo vram`
   - macOS: Apple Silicon unified memory (`sysctl hw.memsize`).
3. **CPU Physical Cores:**
   - Windows: `(Get-CimInstance Win32_Processor).NumberOfCores`
   - Linux: `nproc --all` / `lscpu | grep "Core(s) per socket"`
   - macOS: `sysctl -n hw.physicalcpu`
4. **System RAM:** Must have at least ~24 GB+ RAM to comfortably hold offloaded experts + conversation cache for `UD-Q3_K_XL` (13.63 GB) on an 8 GB GPU.

---

## Step 2: Install PolyStrata & Download Inkling 2 Mini

1. Clone and set up **PolyStrata** (`https://github.com/simplexai-labs/PolyStrata`) following its README (`polystrata.py` / `terminal/setup.py`), or update an existing PolyStrata install (`git pull`).
2. Download an **`inkling2_loop`-layout** GGUF (Note: `strata-poly` requires the `inkling2_loop` layout, e.g., `Mainline_Support` in `unsloth/Inkling-2-Mini-GGUF` or an `inkling2_loop` abliterated quant like `Inkling-2-Mini-Abliterated-Balanced-UD-Q3_K_XL.gguf`, plus `mmproj-BF16.gguf` if vision is desired).
3. Run PolyStrata's setup/pack generator to generate `expert-profile.txt` and `tokenizer/` (verify `tokenizer/tokenizer.json` has `"pre": "inkling2"`).

---

## Step 3: Build `strata-inkling.json` From Template

Start from `configs/strata-inkling.template.json` and replace:
- `<POLYSTRATA_DIR>`, `<MODEL_GGUF_PATH>`, `<MMPROJ_PATH>`, `<EXPERT_PROFILE_PATH>`, `<TOKENIZER_DIR>`, `<PYTHON_LIB_DIRS>` with actual local paths.
- Set `--threads` to the machine's **physical core count** (e.g., `16` on a 16C/32T CPU; avoid SMT oversubscription).
- Set OS/GPU-specific flags:
  - **Windows + NVIDIA 8–12 GB GPU:** Keep `--kv fp8 --vram-pool --vram-reserve-mib 640 --lock-experts --prompt-cache 6 --conversation-cache-mib auto --loop-bf16 --window 8`. (Raise `--vram-reserve-mib` to `768` if desktop apps/browsers use >1.5 GB VRAM.)
  - **Linux + NVIDIA 8–12 GB GPU:** Same as above, with `--vram-reserve-mib 256` to `384`.
  - **16 GB+ NVIDIA GPU:** Same, with `--vram-reserve-mib 256` (or `512` on Windows).
  - **macOS Apple Silicon (Metal beta):** Remove `--vram-pool`, `--vram-reserve-mib`, `--lock-experts`, and `--kv fp8`. Keep `--max-context`, `--prompt-cache 6`, `--conversation-cache-mib auto`, `--window 8`, and `--threads <physical_cores>`.
- **Do NOT add** `--tail-skip 4`, `--attn-requant q5_0`, `STRATA_FAST_ATTN=1`, `STRATA_IQ3_LANES=1`, or `STRATA_IQ_SLOT_LANES=1` (they degrade accuracy and can trigger repetition loops on 8 GB GPUs).
- Keep `"host": "0.0.0.0"` + `"api_key": "none"` if serving to other devices on LAN/Tailscale (or `"127.0.0.1"` for local-only).

---

## Step 4: Check & Apply PolyStrata Server Fixes

Read `patches/POLYSTRATA_FIXES.md` and check `terminal/serve/frontend.py` and `terminal/serve/server.py` in the PolyStrata repo:
1. Ensure `InklingOutputParser._call()` supports `lenient_json()`, markdown fence stripping, stringified args, `"parameters"`/`"input"`/unwrapped args, and `<|content_model_end_sampling|>` call termination.
2. Ensure `InklingOutputParser._read()` `header` mode tolerates leading whitespace before tool names.
3. Ensure `reasoning_close_retry` uses `INK_END + INK_MSG` for `call_format == "inkling"`.
4. Ensure `server.py` treats `"illegal memory access"` in engine `ERR` lines as fatal so a poisoned CUDA process auto-restarts on the next request.
5. Run `python -m unittest discover -s terminal/serve -p "test_inkling.py"` (`PYTHONPATH=terminal`) to confirm 0 failures.

---

## Step 5: Configure Coding Agents

### OpenCode (`~/.config/opencode/opencode.jsonc`)
- Merge `configs/opencode.snippet.jsonc` under `"provider.polystrata"`.
- Ensure `"limit": { "context": 262144, "output": 65536 }`, `"reasoning": true`, and `"tool_call": true` are set.
- Tip: Disable unused MCP servers in `opencode.jsonc` (`"enabled": false`) so idle MCP tool schemas don't add 30K+ tokens to every Turn-1 prompt.

### Oh My Pi (`omp`) (`~/.omp/agent/`)
- Merge `configs/omp-config.snippet.yml` into `~/.omp/agent/config.yml` (crucial: `edit.modelVariants: { inkling: replace }` switches Inkling off `hashline` onto clean `replace` edits).
- Merge `configs/omp-models.snippet.yml` into `~/.omp/agent/models.yml` (`maxTokens: 65536`).

---

## Step 6: Start & Verify

1. Start PolyStrata:
   ```bash
   python terminal/serve/server.py --engine strata --config strata-inkling.json --port 8080
   ```
2. Wait for `http://127.0.0.1:8080/health` to return `"loaded": true`.
3. Send a quick `/v1/chat/completions` test with `model: "inkling-2-mini"`, `reasoning_effort: "high"`, and verify:
   - Non-empty coherent response, `0` thinking loops
   - Tool-call test (`edit` or `question` schema) returns `finish_reason: "tool_calls"` with valid JSON arguments.
4. On Windows, if connecting from another device on LAN/Tailscale, open TCP port `8080` in an **Admin PowerShell**:
   ```powershell
   New-NetFirewallRule -DisplayName "PolyStrata 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow -Profile Private
   ```
