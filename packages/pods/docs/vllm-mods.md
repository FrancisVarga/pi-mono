# vLLM Mods in `pi`: How They Work and How They Were Implemented

This report explains the practical "vLLM modifications" in `pi` (`packages/pods`): what is modified compared to a plain `vllm serve` setup, where those modifications live in code, and how the end-to-end implementation works.

---

## 1) What "vLLM mods" means in this repository

`pi` does not fork vLLM source code. Instead, it implements **operational and runtime modifications around vLLM**:

- Installation-mode variants (`release`, `nightly`, `gpt-oss`) with different wheel/index strategies.
- Model-specific launch profiles that inject vLLM arguments and environment variables.
- Automatic tool-calling parser selection for known models.
- GPU-aware config selection and process launch orchestration.
- Guardrails and defaults for usage tracking, memory behavior, and model cache storage.

So the "mods" are implemented as **deployment scripts + TypeScript configuration/command assembly**, not by patching Python internals of vLLM.

---

## 2) Implementation map (files and responsibilities)

### Core bootstrap and runtime scripts

- `packages/pods/scripts/pod_setup.sh`
  - Installs Python/uv/CUDA alignment and vLLM.
  - Handles `--vllm release|nightly|gpt-oss`.
  - Persists vLLM-related environment settings into `~/.bashrc`.

- `packages/pods/scripts/model_run.sh`
  - Runtime launcher template for each model process.
  - Builds final `vllm serve ...` command from placeholders and injected args.
  - Downloads model (`hf download`) and starts vLLM in a monitored process.

### Config and orchestration in TypeScript

- `packages/pods/src/models.json`
  - Source of truth for known model profiles.
  - Defines per-model/per-GPU-count `args`, optional `env`, and notes.

- `packages/pods/src/model-configs.ts`
  - Reads `models.json`.
  - Picks best config by requested GPU count and GPU type matching.

- `packages/pods/src/commands/models.ts`
  - Implements `pi start`, `pi stop`, `pi list`, `pi logs`.
  - Selects GPUs, applies model config, assembles vLLM args/env, uploads and executes launcher script.
  - Watches logs for success/failure patterns.

### Reference docs that describe these behaviors

- `packages/pods/README.md`
- `packages/pods/docs/models.md`
- `packages/pods/docs/gpt-oss.md`
- `packages/pods/docs/qwen3-coder.md`
- `packages/pods/docs/gml-4.5.md`
- `packages/pods/docs/kimi-k2.md`

---

## 3) Setup-time mod: vLLM installation channels

Implementation location: `packages/pods/scripts/pod_setup.sh` (case block on `VLLM_VERSION`).

`pi pods setup ... --vllm <variant>` changes how vLLM is installed:

### `release` (default)

- Installs stable vLLM track:
  - `uv pip install vllm>=0.10.0 --torch-backend=auto`
- Goal: predictable baseline for most users.

### `nightly`

- Installs latest nightly wheels:
  - `uv pip install -U vllm --torch-backend=auto --extra-index-url https://wheels.vllm.ai/nightly`
- Goal: access to newest model support/features before release.

### `gpt-oss`

- Installs special GPT-OSS build:
  - `vllm==0.10.1+gptoss` via `https://wheels.vllm.ai/gpt-oss/`
  - Additional PyTorch nightly index URL derived from detected CUDA (`cuXYZ`).
- Also installs `gpt-oss` helper package for tool support.
- Goal: dedicated compatibility path for OpenAI GPT-OSS models.

### Why this is a mod

Vanilla setups typically install one vLLM channel manually. `pi` codifies channel-specific behavior into one deterministic setup command.

---

## 4) Runtime mod: standardized command templating and launch

Implementation locations:

- Template: `packages/pods/scripts/model_run.sh`
- Injection/orchestration: `packages/pods/src/commands/models.ts`

The launch path is:

1. `models.ts` computes runtime values:
   - `MODEL_ID`, `NAME`, `PORT`, `VLLM_ARGS`
2. It reads `model_run.sh`, replaces placeholders, and uploads script to pod.
3. Script activates venv, downloads model cache if needed, then runs:
   - `vllm serve '<model>' --port <port> --api-key '<key>' <injected args>`
4. `models.ts` tails logs and marks success when startup text appears.

### Why this is a mod

Instead of forcing users to handcraft each `vllm serve` command, `pi` creates a controlled command-generation pipeline with reproducible defaults.

---

## 5) Model-profile mod: parser, parallelism, env tuning

Implementation location: `packages/pods/src/models.json` with selector in `model-configs.ts`.

For known models, `pi` injects curated vLLM arguments, for example:

- Tool parsers:
  - `--tool-call-parser hermes`
  - `--tool-call-parser qwen3_coder`
  - `--tool-call-parser glm45`
  - `--tool-call-parser kimi_k2`
- Tool auto-choice:
  - `--enable-auto-tool-choice`
- Reasoning parser where relevant:
  - `--reasoning-parser glm45`
- Parallelism/scale settings:
  - `--tensor-parallel-size ...`
  - `--data-parallel-size ...`
  - `--enable-expert-parallel`
- Model-specific env tuning:
  - `VLLM_USE_DEEP_GEMM=1`
  - `VLLM_ATTENTION_BACKEND=XFORMERS`
  - GPT-OSS-specific acceleration flags (documented in `docs/gpt-oss.md`)

### Config resolution algorithm

In `model-configs.ts`, config selection is:

1. Find known model entry by exact model ID.
2. Match config by requested GPU count.
3. If config restricts GPU types, validate against detected GPU name.
4. If no GPU-type match, fallback to same GPU-count config.
5. Return `args` + optional `env`.

### Why this is a mod

This turns vLLM from "raw CLI options per run" into a maintained profile catalog for agentic workloads.

---

## 6) Endpoint/tooling behavior mod (not just flags)

Implementation is split between config and docs:

- GPT-OSS is treated specially: tool use is documented around `/v1/responses` rather than relying on chat-completions function-calling parity.
- Other known models use chat-completions-compatible parser flags via model profiles.

This gives users model-family-aware behavior without requiring them to memorize parser/endpoint constraints.

---

## 7) Environment and safety defaults

Implementation location: `packages/pods/scripts/pod_setup.sh` (`~/.bashrc` append section).

Persistent defaults include:

- `VLLM_NO_USAGE_STATS=1`
- `VLLM_DO_NOT_TRACK=1`
- `VLLM_ALLOW_LONG_MAX_MODEL_LEN=1`
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
- Token/cache wiring:
  - `HF_TOKEN`, `HUGGING_FACE_HUB_TOKEN`
  - `HF_HUB_ENABLE_HF_TRANSFER=1`
- Model cache relocation to configured storage path via `~/.cache/huggingface` symlink.

### Why this is a mod

These defaults convert one-off local shell tweaks into automated, persistent deployment behavior for remote pods.

---

## 8) GPU-aware orchestration mod

Implementation location: `packages/pods/src/commands/models.ts`.

Key behavior:

- Detects current GPU inventory (`nvidia-smi` output parsed during setup).
- Selects GPUs based on least-used assignment for balancing.
- Applies single-GPU pinning via `CUDA_VISIBLE_DEVICES` when appropriate.
- Supports multi-GPU profiles (tensor/data parallel) from model configs.
- Tracks running models with metadata (`name`, model ID, port, gpu IDs, pid).

### Why this is a mod

`pi` adds an orchestration layer around vLLM so multiple model servers can coexist on one pod with predictable resource assignment.

---

## 9) Failure handling and observability mods

Implementation location: `packages/pods/src/commands/models.ts` and `model_run.sh`.

Behavior:

- Startup log monitoring detects:
  - Successful startup marker.
  - Common failure classes (OOM, engine init errors, process exit).
- Logs are persisted under `~/.vllm_logs`.
- `pi logs <name>` streams startup/runtime logs.
- `pi list` reports model process/health state.

This is an operational layer that plain `vllm serve` does not provide by default.

---

## 10) End-to-end flow: how all mods combine

1. User runs `pi pods setup ... --vllm <variant>`.
2. Setup script prepares CUDA/venv/vLLM + persistent environment defaults + model cache path.
3. User runs `pi start <model> --name <name> [--gpus ... | --vllm ...]`.
4. Orchestrator (`models.ts`) resolves known model profile or custom args.
5. Launch script template is filled and executed on pod.
6. vLLM starts with profile-specific parser/parallelism/env settings.
7. `pi` tracks process health and exposes logs/status commands.

This is the implementation of "vLLM mods" in `pi`: a structured deployment system over stock vLLM.

---

## 11) Design outcome

The implementation gives three practical outcomes:

- **Repeatability**: setup and launch are scripted, not ad hoc.
- **Model-specific correctness**: parser/parallelism/env details live in curated profiles.
- **Operational ergonomics**: users interact with `pi` commands instead of managing many low-level vLLM details manually.
