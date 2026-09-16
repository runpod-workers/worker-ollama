# Ollama Serverless Worker

[![Runpod](https://api.runpod.io/badge/lukepiette/worker-ollama)](https://console.runpod.io/hub/lukepiette/worker-ollama)

Run any [Ollama](https://ollama.com) model or Hugging Face GGUF repo on Runpod Serverless. Works with [Runpod model caching](https://docs.runpod.io/serverless/endpoints/manage-endpoints#model-caching), chat and completion requests, streaming, tool calling, and model caching on network volumes.

## Quickstart

1. Deploy this template from the Runpod Hub
2. Pick a model — either `HF_MODEL` for a Hugging Face GGUF repo or `OLLAMA_MODEL` for an Ollama model (see [Choosing a model](#choosing-a-model)). Leave both empty and you get `llama3.2:3b`.
3. Send a request:

```bash
curl -X POST "https://api.runpod.ai/v2/<ENDPOINT_ID>/runsync" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "messages": [{"role": "user", "content": "Why is the sky blue?"}]
    }
  }'
```

## Choosing a model

> [!IMPORTANT]
> **`Hugging Face GGUF Repo` (`HF_MODEL`) is not the same input as `Model` (`OLLAMA_MODEL`).**
>
> - **`Model`** (`OLLAMA_MODEL`) takes an *Ollama* model name: `llama3.2:3b`, or `hf.co/<repo>:<quant>` to use Ollama's own Hugging Face puller.
> - **`Hugging Face GGUF Repo`** (`HF_MODEL`) takes a *Hugging Face repo id*: `unsloth/Qwen3-8B-GGUF`. This is the input that works with Runpod's model store.
>
> **When both are set, `HF_MODEL` wins and `OLLAMA_MODEL` is ignored entirely.** The worker log says so at startup.
>
> Full precedence, highest first: `input.model` (per request) → `HF_MODEL` + `HF_QUANTIZATION` → `OLLAMA_MODEL` → `llama3.2:3b` when all of them are empty.

### Hugging Face GGUF repos

| Input | Required | Behaviour |
|---|---|---|
| `HF_MODEL` | — | Repo id, e.g. `unsloth/Qwen3-8B-GGUF`. Also accepts `hf.co/<org>/<repo>`, a full `huggingface.co` URL, and a trailing `:<quant>` tag — all normalised to a bare repo id. Must contain `.gguf` files; a safetensors-only repo fails with an error listing what was found. |
| `HF_QUANTIZATION` | no | Matched against filenames on `-`, `_`, `.` and `/` boundaries, case-insensitive. `Q4_K_M` does not match `Q4_K_S`, and `Q4` does not match `Q4_K_M`. No match or an ambiguous match → error listing every quantization in the repo. **Leave it empty and the smallest GGUF in the repo is used.** |
| `HF_MODEL_FILE` | no | Exact filename, e.g. `Qwen3-8B-Q4_K_M.gguf`. Overrides `HF_QUANTIZATION`. |

```
HF_MODEL        = unsloth/Qwen3-8B-GGUF
HF_QUANTIZATION = Q4_K_M
```

> [!TIP]
> **Use the repo's exact casing.** Hugging Face resolves a lowercased repo id with a `307` redirect, so downloads work either way — but Runpod's model store prefills under the canonical casing. The worker falls back to a case-insensitive lookup and logs when it has to, though matching the casing yourself avoids the scan entirely.

If you omit `HF_QUANTIZATION` and the repo id carries no `:<quant>` tag, the worker picks **`Q4_K_M`** — the same default Ollama's own puller uses, and what model cards assume:

```
HF_QUANTIZATION not set — defaulting to Q4_K_M in 'unsloth/Qwen3-8B-GGUF': Qwen3-8B-Q4_K_M.gguf (4.7 GiB). Available quantizations: [...]
```

Only if the repo has no `Q4_K_M` does it fall back to the smallest file, and it says so. That fallback matters because large repos start very low: `unsloth/Qwen3-8B-GGUF`'s smallest is `UD-IQ1_S` at 2.3 GB against Q4_K_M's 5.0 GB, and 1-bit output is not usable for most work.

Set `HF_QUANTIZATION` explicitly whenever you care about the quality/VRAM trade-off.

The model is registered with Ollama as `hf/<org>-<repo>:<quant>` — for the example above, `hf/unsloth-qwen3-8b-gguf:q4_k_m`. That name shows up in `/api/tags` and can be passed as `input.model` on a request.

Multi-part GGUFs (`model-00001-of-00003.gguf`) are supported: name any shard in `HF_MODEL_FILE`, or just set `HF_QUANTIZATION`, and every shard is loaded.

### Ollama models

`OLLAMA_MODEL` accepts anything `ollama pull` accepts:

| Source | Example |
|---|---|
| Ollama library | `llama3.2:3b`, `qwen2.5-coder:7b` |
| Ollama-style HuggingFace reference | `hf.co/prism-ml/Bonsai-27B-gguf:Q1_0` |

For HuggingFace repos referenced this way, specify the quant as a tag (`:Q4_K_M`, `:Q1_0`, `:F16`, ...). Without a tag, Ollama defaults to `Q4_K_M` and fails if the repo doesn't include one. This path does **not** use Runpod's model store — use `HF_MODEL` for that.

**Sizing tip:** the VRAM needed for weights is roughly the size of the GGUF file plus ~15% overhead for KV cache and activations. Pick a GPU with headroom above that.

**Default hardware:** the Hub listing defaults to the 80 GB and 96 GB pools (`BLACKWELL_96`, `ADA_80_PRO`, `AMPERE_80`, i.e. RTX PRO 6000, H100 and A100). That is deliberately generous: the model inputs above invite 27B+ GGUFs, and a 24 GB card silently spills those to CPU, which turns the first request into a multi-minute load that usually times out. Pick a smaller pool on the endpoint if you know your model fits.

## Runpod model caching

Set the endpoint's **Model** field to the *same* repo as `HF_MODEL`. Runpod then pre-downloads it to `/runpod-volume/huggingface-cache/hub/models--<org>--<name>/snapshots/<hash>/` **before the worker starts**, and doesn't bill you for the download. The worker finds it there and registers it with Ollama without fetching anything.

You'll see this in the worker log:

```
[ModelStore] Using snapshot /runpod-volume/huggingface-cache/hub/models--unsloth--Qwen3-0.6B-GGUF/snapshots/<hash>
```

If the model isn't cached, the worker warns and downloads from Hugging Face instead — which *is* billed cold-start time:

```
WARN: no cached snapshot for 'unsloth/Qwen3-0.6B-GGUF' under /runpod-volume/huggingface-cache/hub.
      Downloading from Hugging Face instead (billed cold-start time).
      To use Runpod's model store, set the endpoint's Model field to 'unsloth/Qwen3-0.6B-GGUF'.
```

Caveats worth knowing before you pick a repo:

- **Every quantization in the repo is downloaded.** Runpod can't select one, so a repo with a dozen quants caches all of them — for a large model that can be hundreds of GB of prefill. Prefer repos that ship a single quant when the model is large.
- **One cached model per endpoint.**
- **Gated models need two separate tokens.** The one for the pre-download goes in the console's Model Caching settings; the container's `HF_TOKEN` is only used when the worker has to fetch the model itself.
- The cache path is `/runpod-volume/huggingface-cache`, **not** `/runpod/model-store/`.

## API

### Input

| Field | Type | Required | Description |
|---|---|---|---|
| `messages` | array | one of `messages`/`prompt` | Chat messages, OpenAI format |
| `prompt` | string | one of `messages`/`prompt` | Raw completion prompt |
| `model` | string | no | Overrides the endpoint's configured model; pulled on demand if missing |
| `stream` | bool | no | Stream response chunks (default `false`) |
| `options` | object | no | Ollama options (`temperature`, `num_ctx`, `top_p`, ...) |
| `tools` | array | no | Tool definitions for models that support tool calling |
| `format` | string/object | no | `"json"` or a JSON schema for structured output |
| `system` | string | no | System prompt (completion mode) |
| `template` | string | no | Override the model's chat template for this request |
| `keep_alive` | string/int | no | How long to keep the model loaded (default: forever) |

### Chat request

```json
{
  "input": {
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Write a haiku about GPUs."}
    ],
    "options": {"temperature": 0.7}
  }
}
```

### Completion request

```json
{
  "input": {
    "prompt": "The capital of France is",
    "options": {"num_predict": 10}
  }
}
```

### Streaming

Set `"stream": true` and use `/run` + `/stream/<JOB_ID>`, or `/runsync` to receive the aggregated stream. Each chunk is a JSON line in Ollama's [streaming format](https://github.com/ollama/ollama/blob/main/docs/api.md).

### Output

Non-streaming responses return Ollama's native response object:

```json
{
  "model": "llama3.2:3b",
  "message": {"role": "assistant", "content": "..."},
  "done": true,
  "eval_count": 42,
  "eval_duration": 1234567890
}
```

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `HF_MODEL` | — | Hugging Face GGUF repo id. **Takes precedence over `OLLAMA_MODEL`.** |
| `HF_QUANTIZATION` | smallest in repo | Which GGUF quantization to load (`Q4_K_M`, `Q8_0`, `IQ4_XS`, ...) |
| `HF_MODEL_FILE` | — | Exact `.gguf` filename; overrides `HF_QUANTIZATION` |
| `HF_TOKEN` | — | Hugging Face token for gated/private repos, used only when the worker downloads the model itself |
| `OLLAMA_MODEL` | — | Ollama model pulled at worker startup; ignored when `HF_MODEL` is set |
| — | `llama3.2:3b` | Used when **both** `HF_MODEL` and `OLLAMA_MODEL` are empty |
| `OLLAMA_MODELS` | auto | Ollama's model store. Defaults to `/runpod-volume/ollama/models` when that is writable, else `/root/.ollama/models` |
| `RUNPOD_MODEL_CACHE_DIR` | `/runpod-volume/huggingface-cache/hub` | Where Runpod's model store mounts its cache |
| `OLLAMA_TEMPLATE` | — | Chat template override used when registering a Hugging Face GGUF |
| `OLLAMA_KEEP_ALIVE` | `-1` (forever) | How long models stay loaded in VRAM |
| `OLLAMA_LOAD_TIMEOUT` | `60m` | How long Ollama waits for a model to load into memory before failing the request. Ollama's own default is `5m`, which a large model on a fresh worker can exceed |

## Storage and disk sizing

| Configuration | Ollama store | GGUF acquisition | Disk needed |
|---|---|---|---|
| Writable network volume (**recommended**) | `/runpod-volume/ollama/models` | hard-linked from the cache, no copy | image only, plus ~3× model on the volume |
| Model store, no network volume | `/root/.ollama/models` | copied (different filesystem) | ~3× model + ~5 GB image |
| Neither | `/root/.ollama/models` | downloaded, then hard-linked | ~3× model + ~5 GB image |

> [!IMPORTANT]
> **Budget ~3× the model size at peak.** Ollama re-writes a GGUF when registering it (`validating GGUF model` in the log), so at peak the blob store holds the hard-linked original, a `COPY` temp, and the final blob at once. Measured on a real import: **2.49× sampled, up to 3× transient**, settling to 2× afterwards.
>
> A 20 GiB model therefore needs ~64 GiB free, which is why the default container disk is **100 GB**. The worker now fails fast with the required and available figures instead of letting Ollama die with an opaque 500 after writing 20 GiB.

Where possible the worker hard-links the GGUF into Ollama's blob store rather than uploading it through localhost HTTP, which removes one full copy from the peak.

If you need 1× disk instead, skip the Hugging Face path and use Ollama's native puller — `OLLAMA_MODEL=hf.co/<org>/<repo>:<quant>` fetches pre-built layers with no re-write. You lose model-store caching in exchange.

A writable network volume is what makes cold starts fast: the registered model persists there, so later workers skip the download, the hashing and the registration entirely.

Everything Ollama pulls lands in `OLLAMA_MODELS`, not just model-store models — plain Ollama library pulls (`llama3.2:3b`) and gated `hf.co/...` pulls are cached on the volume too.

The worker can't tell a network volume from a local volume disk from a model-store mount: all three appear at `/runpod-volume`. It reports what it can actually observe at startup:

```
Volume: /runpod-volume present and writable — network volume or local volume disk (indistinguishable from here). Models cached here only survive across workers if it is a network volume.
Model store: /runpod-volume/huggingface-cache/hub present (Runpod pre-downloaded cache)
Ollama store: /runpod-volume/ollama/models
```

If `/runpod-volume` exists but isn't writable — a model-store mount with no volume attached — both caches fall back to container disk and the log says so.

## Limitations

- **GGUF only.** Safetensors-only Hugging Face repos are rejected with an error — use a GGUF conversion of the model.
- **Multimodal projectors are skipped.** A repo shipping `mmproj-*.gguf` alongside the model has that file excluded from automatic selection, since it holds no language-model weights and Ollama rejects every request against it. Vision input is therefore not wired up; the language model is served text-only.
- **One cached model per endpoint**, and the model store downloads every quantization in the repo.
- A GGUF with no embedded chat template produces malformed chat output. Set `OLLAMA_TEMPLATE`, or pass `template` per request. The worker logs a warning when it detects this.
- **VRAM is not enforced, only reported.** If the weights don't fit, the worker warns and Ollama offloads the remainder to CPU — the model still answers, much more slowly. Pick a larger GPU, a smaller `HF_QUANTIZATION`, or a lower `OLLAMA_CONTEXT_LENGTH`.

## Local development

```bash
docker build -t ollama-worker .

# Ollama model
docker run --gpus all -e OLLAMA_MODEL=llama3.2:1b ollama-worker

# Hugging Face GGUF repo
docker run --gpus all -e HF_MODEL=unsloth/Qwen3-0.6B-GGUF -e HF_QUANTIZATION=Q4_K_M ollama-worker
```

Without a GPU, Ollama falls back to CPU inference — slow, but enough to smoke-test the handler with a small model. You can also test the handler directly against `test_input.json`:

```bash
python3 handler.py --rp_serve_api  # requires a local ollama serve
```

The model-selection and naming logic is pure and covered by unit tests that need no GPU, no network and no Ollama:

```bash
python3 -m pytest test_selection.py -v
```
