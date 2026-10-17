# `agent-model-config` — External Model Configuration Repository

This repository is the **control plane** for the multi-agent debate swarm. It is intentionally
tiny: it contains no code and no binaries. The execution repository
(`Rishi794/Github_Agent_Swarm`) checks it out at runtime and reads `model_config.json`.

Editing this file lets you hot-swap the GGUF model, quantization level, `llama-server`
CPU flags, RAG depth, or number of debate rounds **without touching the execution codebase**.

---

## `model_config.json` reference

| Key | Type | Purpose |
| --- | --- | --- |
| `hf_repo` | string | Hugging Face repository that hosts the GGUF weights. |
| `hf_file` | string | Exact GGUF filename (must be a single-file, not sharded, quant). |
| `model_alias` | string | Name the model is served under by `llama-server --alias`; the Python clients pass this as the OpenAI `model` value. |
| `embedding_model` | string | `fastembed` model used for the SQLite vector index (RAG). |
| `embedding_dim` | int | Vector width. **Must match the embedding model** and the existing `vec0` table. |
| `debate_rounds` | int | Number of critique/rebuttal rounds. Round 1 = independent thesis, round N = critique of round N-1. |
| `rag_top_k` | int | How many historical findings are injected into the prompt. |
| `peer_timeout_sec` | int | How long an agent waits for the peer-sync target before proceeding with whichever peers published (it fails only when no peer published at all). |
| `peer_sync_count` | int | How many peers a round-2 agent synchronizes with. `0` = strict all-peers barrier (small rosters); `8` lets 100-agent rosters work despite the account's ~20 concurrent-runner limit. |
| `llm_timeout_sec` | int | Client-side timeout for a single LLM request. Cold 32k-ctx CPU prefill can run for many minutes; the OpenAI SDK default (600 s) was cutting round 1 short. Env override: `LLM_TIMEOUT_SEC`. |
| `enable_tools` | bool | Master switch for the agent tool loop (`search_web`, `query_arxiv`, `fetch_url`, `compute`). |
| `max_tool_iterations` | int | Tool-call budget per debate round, per agent. Each iteration may dispatch one or more tools. |
| `llama_cpp_release` | string | `"auto"` = newest `llama.cpp` nightly with Linux x64 CPU binaries; or pin e.g. `"b11707"`. |
| `llama_server_flags` | object | CPU/RAM tuning flags, translated into `llama-server` CLI arguments. |

### `llama_server_flags` → CLI mapping

| JSON key | CLI flag | Value | Why |
| --- | --- | --- | --- |
| `threads` | `-t` | `4` | Matches the 4 vCPU allocation of `ubuntu-latest`; avoids oversubscription stalls. |
| `threads_batch` | `-tb` | `4` | Threads for prompt evaluation (batch) processing. |
| `ctx_size` | `-c` | `32768` | Context window per slot. Must exceed `system_prompt.md` (~13k tokens for the current charter) plus retrieved memory, peer findings and tool results. 32k KV ≈ 2.5 GB at `q8_0` on Qwen3-8B. |
| `batch_size` | `-b` | `512` | Logical prompt batch size. |
| `ubatch_size` | `-ub` | `256` | Physical micro-batch; keeps the compute scratchpad inside L3. |
| `parallel_slots` | `-np` | `1` | Single sequence slot: cuts KV-cache allocation ~4x vs. the multi-slot default. |
| `cache_type_k` | `-ctk` | `q8_0` | 8-bit KV-cache keys. |
| `cache_type_v` | `-ctv` | `q8_0` | 8-bit KV-cache values. |
| `load_mode` | `-lm` | `mmap` | Kernel can page tensor blocks out under memory pressure (no `mlock`). |
| `flash_attn` | `-fa` | `on` | Fused attention kernels; required for quantized V-cache. |

---

## Common upgrades

**Switch to a 14B dense model (still fits 16 GB standard runners):**

```json
"hf_repo": "Qwen/Qwen3-14B-GGUF",
"hf_file": "Qwen3-14B-Q4_K_M.gguf",
"model_alias": "qwen3-14b-pareto"
```

**MoE model on a 32 GB+ self-hosted runner** (frontier quality at 3B–4B active-parameter speed):

```json
"hf_repo": "Qwen/Qwen3.5-35B-A3B-GGUF",
"hf_file": "Qwen3.5-35B-A3B-Q4_K_M.gguf"
```

**Faster prompt ingestion / longer debates:**

```json
"debate_rounds": 3,
"rag_top_k": 6,
"llama_server_flags": { "batch_size": 1024, "ubatch_size": 512 }
```

### Sizing `ctx_size` to the mission

Every agent prepends the **full contents of `system_prompt.md`** to every round, so the
window has to hold:

```
ctx_size  ≥  tokens(system_prompt.md) + RAG context + peer findings + tool results + answer
             ~13 000 (current charter)    ~1 500        ~2 000        up to ~6 000    ~1 000
```

Rule of thumb: **tokens ≈ bytes / 4**. The current charter is ~53 KB ≈ 13 000 tokens, so
`ctx_size: 32768` leaves real headroom. Shrink the mission and you can shrink the window;
grow it and raise `ctx_size` with it. The workflow prints the resolved prompt size and
warns when it exceeds ~20 000 tokens.

KV cache at `q8_0` on Qwen3-8B costs roughly **78 KB per token**:

| `ctx_size` | KV cache | Total with 4.7 GB model + ~1 GB buffers |
| --- | --- | --- |
| 8 192 | ~0.6 GB | ~6.5 GB |
| 32 768 | ~2.5 GB | ~8.5 GB |
| 65 536 | ~5.1 GB | ~11 GB |

All three fit a 16 GB runner; a bigger window mainly costs prefill time, not safety.

> **Rule of thumb:** keep `(model size + KV cache + ~1 GB Python/OS overhead)` below the runner's
> RAM. On a 16 GB `ubuntu-latest` runner, stay at or below a ~9 GB GGUF file with `-np 1` and
> 8-bit KV cache. Anything larger will be `SIGKILL`ed (exit code 137).

---

## Tools the agents can call

Tool calling is native (`llama-server --jinja`), so the model invokes real Python
functions mid-reasoning instead of guessing. Each persona in
`agents/system_prompts.json` declares which tools it may use.

| Tool | What it does |
| --- | --- |
| `search_web(query, max_results)` | Live web search (DuckDuckGo HTML → DuckDuckGo Instant Answer → Wikipedia fallback chain). |
| `query_arxiv(search_query, max_results)` | arXiv API: titles, authors, dates, abstracts, links. |
| `fetch_url(url, max_chars)` | Reads a specific page as text (SSRF-guarded, size-capped). |
| `compute(expression)` | AST-restricted arithmetic for exact math (`sqrt`, `log`, `**`, `pi`, …). |

Tuning knobs live here, not in code:

```json
"enable_tools": true,
"max_tool_iterations": 3
```

* `enable_tools: false` — run a pure-parametric debate (fastest, no network egress).
* `max_tool_iterations: 1` — cheap groundings only: one tool round before answering.
* `max_tool_iterations: 6` — deep research rounds; slower on 4 vCPU runners, and more
  prompt tokens per round.

Tool results are truncated to 6 000 characters per call (~1 500 tokens) so a large
page cannot crowd out the debate itself. Override with the `MAX_TOOL_RESULT_CHARS`
environment variable in the workflow if you raise `ctx_size` further.

---

## Access model

This repository is **public**, so the swarm workflow reads it with the default `GITHUB_TOKEN`
and **no secret is required**.

If you fork this into a private repository, create a fine-grained PAT with read-only
`Contents` permission and store it in the execution repository as the secret
`CONFIG_REPO_TOKEN`. The workflow already prefers that secret when present:

```yaml
token: ${{ secrets.CONFIG_REPO_TOKEN || github.token }}
```
