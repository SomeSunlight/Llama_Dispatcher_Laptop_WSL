# Llama_Dispatcher – Instance: Laptop

This repository contains the versioned configuration of the **Laptop** instance for [Llama_Dispatcher](https://github.com/SomeSunlight/Llama_Dispatcher). The same configuration is intended to be usable from Windows and WSL; environment-local runtime paths and generated data stay outside the repository.

## Content

| Directory / File | Description |
|---|---|
| `instance.yaml` | Machine GUID and nickname of this instance |
| `profiles/` | YAML profiles with model-relative paths and llama.cpp parameters |
| `ensembles/` | Backend-specific model/alias compositions |
| `engines/` | Hardware/backend policy, including process environment such as GPU selection |

Runtime data deliberately does **not** belong to the instance repository:

- `data/metrics.db` is created locally by the Dispatcher and remains specific to that environment.
- generated router presets are recreated locally.
- SQLite WAL/SHM files are local runtime state as well.

This keeps Windows and WSL runs separate even when they use the same versioned Laptop configuration.

## Portable model paths

Profiles use `${LLAMA_MODEL_ROOT}` instead of embedding `C:\...` or `/mnt/c/...` roots:

```yaml
common:
  m: "${LLAMA_MODEL_ROOT}/gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf"
```

The concrete root is runtime input. A managed launcher such as AI Workstation passes it explicitly with `--model-root`; direct/manual Dispatcher use may use the same CLI option. The optional `LLAMA_MODEL_ROOT` process environment variable is only a fallback supported by Dispatcher, not a prerequisite that must be placed in a shell profile.

## Engine-owned device selection

GPU-selection environment belongs to the engine rather than to startup instructions in each profile:

```yaml
environment:
  GGML_VK_VISIBLE_DEVICES: "0"
```

Dispatcher applies the engine environment when it launches llama.cpp. The caller therefore does not have to remember `$env:GGML_VK_VISIBLE_DEVICES=...` or export it in WSL.

The Laptop currently distinguishes these engines explicitly:

- `vulkan-intel` — Vulkan device `0`;
- `vulkan-rtx500` — Vulkan device `1`;
- `sycl` — Intel Level Zero device selected through `ONEAPI_DEVICE_SELECTOR=level_zero:0`.

This is deliberate hardware policy, not an operating-system path difference. Profiles point to the engine that matches the hardware they are designed for.

## Associated Dispatcher

The Dispatcher itself (code, defaults, documentation) is located in the public repo:
https://github.com/SomeSunlight/Llama_Dispatcher

## Setup in a Dispatcher checkout

The main repo and the instance repo must be cloned into the exact expected directories. `git clone <url> <target_directory>` gives the instance repository the required directory name:

```bash
# 1. Clone main repo
git clone https://github.com/SomeSunlight/Llama_Dispatcher.git
cd Llama_Dispatcher

# 2. Clone this instance into the Dispatcher's user-owned instance area
git clone https://github.com/SomeSunlight/Llama_Dispatcher_Laptop.git instances/Laptop

# 3. Set up Python environment
uv sync
```

The Dispatcher creates `instances/Laptop/data/metrics.db` locally when needed. Do not copy another environment's database unless historical metrics are intentionally being migrated.

`instance.yaml` carries the instance identity. When this configuration is reused in a materially different environment, review the identity before recording benchmark history.

## Windows and WSL

The model files and versioned profile semantics are shared. Machine-local binary and model roots are supplied at process start and are not duplicated in the instance repository.

For WSL/SYCL use the dedicated `thinkpad-sycl` ensemble. The existing `thinkpad` ensemble remains the Intel/Vulkan configuration. Backend-specific differences are kept separate where they are real; filesystem-root differences are not duplicated.

A WSL run should normally keep its own local database even when Windows tests continue in parallel.

## Usage

Direct Dispatcher example:

```bash
uv run src/dispatcher.py serve \
  --ensemble thinkpad-sycl \
  --instance Laptop \
  --bin-dir /path/to/llama.cpp/build/bin \
  --model-root /path/to/models
```

When AI Workstation manages the runtime, configure these local paths once there and use its simple start/stop commands instead of this full invocation.

## License

MIT.
