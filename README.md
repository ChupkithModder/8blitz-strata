<h1 align="center">8blitz-strata</h1>

<p align="center"><b>Run Inkling 2 Mini at 256K context on an 8 GB GPU — fast, lossless, and agent-ready</b><br>
Tuned <a href="https://github.com/simplexai-labs/PolyStrata">PolyStrata</a> (<code>strata-poly</code>) + OpenCode + Oh My Pi (<code>omp</code>) setup for 8 GB+ cards</p>

<p align="center"><a href="https://chupkithmodder.github.io/8blitz-strata/pagoda.html"><img src="assets/pagoda.gif" width="780" alt="Voxel Pagoda Garden generated in one shot by Inkling 2 Mini on an 8 GB RTX 3070"></a><br>
<sub>Inkling 2 Mini (<code>UD-Q3_K_XL</code>) on an 8 GB RTX 3070, 0 thinking loops: writes a Three.js voxel pagoda and cherry blossom garden in one HTML file. <a href="https://chupkithmodder.github.io/8blitz-strata/pagoda.html"><b>Try the live 3D demo in your browser</b></a>.</sub></p>

**8blitz-strata** is a tuned configuration, fix pack, and automated agent runbook for running **[Inkling 2 Mini](https://huggingface.co/thinkingmachines/Inkling-2-Mini)** (`UD-Q3_K_XL`, 13.63 GB) locally on an **8 GB GPU** with a full **256K (`262,144`) native context window** — without PolyStrata's lossy small-card approximations, without thinking loops, and without broken tool calls in coding agents.

> **No model weights or binaries are included in this repo.** Give this repo to your coding agent and it will set everything up for your OS and hardware.

---

## Give this repo to your agent

Paste this into **OpenCode**, **Oh My Pi (`omp`)**, **Claude Code**, **Codex**, or any coding agent:

```text
Clone https://github.com/ChupkithModder/8blitz-strata and follow AGENT_SETUP.md to install and configure Inkling 2 Mini on PolyStrata for my OS, GPU, and CPU, apply the server/agent fixes, and run a verification test.
```

Full step-by-step instructions: **[AGENT_SETUP.md](AGENT_SETUP.md)**

---

## Context window & memory (8 GB VRAM + 48 GB RAM)

Inkling 2 Mini ships with a **262,144-token (256K)** context window in this setup (`--max-context 262144`):

- **`--kv fp8`** stores K/V caches in 8-bit FP8 (`~16 KiB/token` vs `~32 KiB/token` in FP16), growing lazily in 4,096-row pages inside the VRAM pool so short prompts keep maximum experts in VRAM while long coding sessions scale cleanly.
- **`--conversation-cache-mib auto`** parks inactive conversations in system RAM (`~110K` tokens parked/restored in `~319 ms`) so switching sessions doesn't re-read 50K+ tokens.
- **`--vram-reserve-mib 640`** (Windows) keeps Windows WDDM from silently paging VRAM to system RAM over PCIe, which otherwise cuts decode speed in half (`14–25 tok/s`).

| Context Depth | Writes answers | Reads your prompt | Notes |
| --- | ---: | ---: | --- |
| **Short / Edit (`0–2K` tokens)** | **49 – 64.7 tok/s** | **421 tok/s** | `1,041 / 3,072` experts in VRAM, `90.2%` draft acceptance (`--window 8`) |
| **Coding Session (`50K–53K` tokens)** | **30.1 – 42.3 tok/s** | **796 tok/s** | Tested live in OpenCode (`49.9K` Turn-1 + `edit`/`bash`/`read`) |
| **Long Context (`107K–110K` tokens)** | **25.4 – 32.3 tok/s** | **127 – 237 tok/s** | Tested up to `110,840` tokens, zero crashes or OOMs |

---

## How fast is it?

A token is about ¾ of a word.

- **Writes answers:** how fast the reply appears (`60 tok/s` is faster than you can read).
- **Reads your prompt:** how fast it ingests files, tool schemas, and chat history.

All numbers below were measured on an **RTX 3070 8 GB (`sm_86`) + Ryzen 9 5900XT (16C/32T) + 48 GB RAM** on Windows 11 using `Inkling-2-Mini-Abliterated-Balanced-UD-Q3_K_XL.gguf` (`13.63 GB`, `inkling2_loop` layout):

### Engine comparison (same hardware, same GGUF)

| Engine | Writes answers | Reads your prompt | Status on 8 GB RTX 3070 |
| --- | ---: | ---: | --- |
| **PolyStrata (`strata-poly`, tuned)** | **49.0 – 64.7 tok/s** | **421 – 796 tok/s** | `90.2%` speculative draft acceptance (`--window 8 --lookup 3`) |
| **Unsloth `llama.cpp` (`b12676`)** | 5.1 – 8.4 tok/s | 37.6 – 42.3 tok/s | `--fit on` conflicted with `--moe-cache-mib auto`; PCIe bottlenecked decode |
| **Mainline `llama.cpp` (`b12728`)** | — | — | Failed to load (`tensor 'blk.24.attn_q.weight' not found`) |

### Zero-quality-loss tuning (A/B tested, `temperature=0.0`)

All profiles kept `--kv fp8 --vram-pool --lock-experts --prompt-cache 6 --conversation-cache-mib auto --loop-bf16 --lookup 3`:

| Profile | Fresh Code (W1) | Code Edit (W2) | Reads Prompt (W2) | Drafts Accepted |
| --- | ---: | ---: | ---: | ---: |
| `--threads 16 --window 4` | **49.5 tok/s** | 61.6 tok/s | 419.3 tok/s | 90.2% |
| `--threads 22 --window 4` (auto SMT) | 48.8 tok/s | 58.9 tok/s | 419.6 tok/s | 88.4% |
| `--threads 24 --window 4` + `STREAM_STAGE=1` | 45.3 tok/s | 59.8 tok/s | 403.4 tok/s | 91.8% |
| **Winner: `--threads 16 --window 8`** | 49.0 tok/s | **64.7 tok/s** | **421.0 tok/s** | **453 / 502 (90.2%)** |

### Single-file 3D, Game & Frontend benchmark (`reasoning_effort: "high"`, `max_tokens: 32768`)

| Scene / App | Thinking / Answer Tokens | Writes (Think / Answer) | Output Size | `node --check` | Loops |
| --- | ---: | ---: | ---: | --- | --- |
| **Voxel Sky-Island** (Three.js r160) | 6,146 / 16,082 | 50.2 / 46.6 tok/s | 35.9 KB (868 lines) | Pass (`0` errors) | **0** |
| **Neon Cyber Arcade** (Canvas + WebAudio) | 6,146 / 19,210 | 46.7 / 43.9 tok/s | 44.0 KB (1,178 lines) | Pass (`0` errors) | **0** |
| **Synth & Beat Studio** (WebAudio + Spectrum) | 6,146 / 18,202 | 46.4 / 44.5 tok/s | 43.3 KB (949 lines) | Pass (`0` errors) | **0** |
| **Voxel Pagoda** (Three.js r128, GIF above) | 6,146 / 7,341 | 24.4* / 25.8* tok/s | 14.3 KB (410 lines) | Pass (`0` errors) | **0** |

*\*Measured before raising `--vram-reserve-mib` from `256` to `640`, illustrating the WDDM VRAM paging slowdown that `640` fixed.*

---

## What was fixed

1. **Removed lossy small-card flags:** PolyStrata's default `<16 GB` installer adds `--tail-skip 4` (drops up to ~25% of offloaded CPU expert computations on 8 GB cards without renormalization), `--attn-requant q5_0`, `STRATA_FAST_ATTN=1`, and unchecked `STRATA_IQ3_LANES` / `STRATA_IQ_SLOT_LANES`. Removing them eliminated repetition loops and restored full accuracy, while `--kv fp8` + `--loop-bf16` preserve `98.1–98.4%` top-1 agreement (`median KL 0.0003`).
2. **Thinking budget without dumbing down:** `reasoning_effort: "high"` + `reasoning_budget_tokens: 6144` keeps full planning depth without burning 12K+ tokens and truncating large HTML/code builds.
3. **PolyStrata parser & crash fixes ([patches/POLYSTRATA_FIXES.md](patches/POLYSTRATA_FIXES.md)):**
   - Added `lenient_json()`, markdown fence stripping, stringified-arg decoding, unwrapped arg fallback, leading-whitespace header tolerance, and `<|content_model_end_sampling|>` termination to `InklingOutputParser` so nested tool calls (like OpenCode's `question` tool) never silently downgrade to plain text (`finish_reason: "stop"`).
   - Fixed `reasoning_close_retry` to emit `<|end_message|><|message_model|>` instead of `</think>`.
   - Auto-terminates poisoned CUDA processes on `illegal memory access` so the next request cleanly respawns.
4. **Coding agent fixes ([configs/](configs/)):**
   - **OpenCode:** `output: 65536`, `tool_call: true`, `reasoning: true`, plus disabling unused MCPs to save ~35K schema tokens on Turn 1.
   - **Oh My Pi (`omp`):** `edit.modelVariants: { inkling: replace }` (switches off `hashline` onto `replace`) and `maxTokens: 65536`.

---

## Cross-OS support

| OS | Backend | Key Adjustments |
| --- | --- | --- |
| **Windows (NVIDIA 8 GB+)** | `strata-poly.exe` (CUDA) | `--vram-reserve-mib 640` (or `768` under heavy desktop/browser GPU load) + `--threads` = physical core count (`16` on 16C/32T). |
| **Linux (NVIDIA 8 GB+)** | `strata-poly` (CUDA) | `--vram-reserve-mib 256`–`384` (lower display overhead than WDDM). Enable HugePages / `memlock` for `--lock-experts`. |
| **Linux (AMD ROCm)** | `strata-poly` (HIP) | Swap `--kv fp8` for `--kv q8_0` if FP8 KV is unsupported on your GPU. |
| **macOS (Apple Silicon)** | `strata-poly` (Metal beta) | Omit `--vram-pool`, `--vram-reserve-mib`, `--lock-experts`, and `--kv fp8` (unified memory; Metal refuses `--kv q8_0` on Inkling). Keep `--window 8` and `--prompt-cache 6`. |

---

## Files in this repo

- [AGENT_SETUP.md](AGENT_SETUP.md) — Automated & manual setup runbook for AI agents
- [configs/strata-inkling.template.json](configs/strata-inkling.template.json) — Sanitized 256K-context PolyStrata config template
- [configs/opencode.snippet.jsonc](configs/opencode.snippet.jsonc) — OpenCode provider snippet
- [configs/omp-config.snippet.yml](configs/omp-config.snippet.yml) & [configs/omp-models.snippet.yml](configs/omp-models.snippet.yml) — Oh My Pi (`omp`) config snippets
- [patches/POLYSTRATA_FIXES.md](patches/POLYSTRATA_FIXES.md) — `frontend.py` & `server.py` fixes
- [pagoda.html](pagoda.html) & [assets/pagoda.gif](assets/pagoda.gif) — Single-file Three.js voxel pagoda demo & low-FPS preview
