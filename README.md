# Llama_Dispatcher – Instance: Laptop_WSL

This repository contains the versioned **WSL-specific** configuration for the `Laptop_WSL` instance of [Llama_Dispatcher](https://github.com/SomeSunlight/Llama_Dispatcher).

Windows is intentionally a separate instance. This repository contains only portable model references and WSL backend/profile policy; machine-local llama.cpp binaries and generated runtime data stay outside Git.

## Content

| Directory / File | Purpose |
|---|---|
| `instance.yaml` | Stable identity of the WSL instance |
| `profiles/` | Model/runtime profiles for Intel SYCL, Intel Vulkan and RTX 500 Vulkan experiments |
| `ensembles/` | Client-facing model aliases and backend-specific compositions |
| `engines/` | Backend/device policy and process environment |

Runtime data is intentionally local:

- `data/*.db` contains Dispatcher metrics for this WSL instance.
- generated `data/*_models.ini` router presets are recreated locally.
- SQLite WAL/SHM files are local runtime state.

## Runtime paths

Profiles use `${LLAMA_MODEL_ROOT}` instead of embedding a Windows or WSL model root:

```yaml
common:
  m: "${LLAMA_MODEL_ROOT}/gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf"
```

AI Workstation normally supplies both concrete runtime roots:

- `--model-root` selects the machine-local GGUF root;
- `--bin-dir` selects the concrete llama.cpp build.

The engines therefore contain **no `bin_dir`**. Their job is backend and device policy, not installation-path ownership.

## Engines

### `sycl`

Intel integrated GPU through SYCL / Level Zero.

```yaml
environment:
  ONEAPI_DEVICE_SELECTOR: "level_zero:0"
  ZES_ENABLE_SYSMAN: "1"
```

The temporary OpenCL path used during Issue #1/#8 diagnosis is not retained as a normal engine. It was a diagnostic control, while Level Zero is the versioned SYCL target.

### `vulkan-intel`

Intel Vulkan path, explicitly selecting Vulkan device 0.

### `vulkan-rtx500`

RTX 500 Ada Vulkan path, explicitly selecting Vulkan device 1.

Vulkan device numbering must still be verified on the actual WSL Vulkan implementation before performance results are treated as accepted evidence.

## Standalone profiles versus ensembles

Network settings are intentionally separated from the model profiles:

- `engines/*.yaml -> serve.host/port` provides the default address for **direct single-profile serve**.
- `ensembles/*.yaml -> engine.host/port` owns the llama.cpp router address in **ensemble mode**.
- profiles therefore contain model/runtime tuning only and can still be started directly without repeating host/port on the command line.

The Dispatcher merge/compile logic keeps these two cases separate: direct profile serve inherits the engine `serve` defaults, while ensemble model sections do not use those host/port values.

## Ensembles

- `thinkpad-sycl` — current Intel SYCL / Level Zero ensemble using `Thinkpad_SYCL_gemma_26B_A4B`.
- `thinkpad-vulkan` — Intel Vulkan ensemble using `Thinkpad_vulkan_gemma_26B_A4B`.

Both expose:

- `sparringpartner` — real loaded model with thinking enabled;
- `agent` — proxy-only alias targeting the same loaded model with thinking disabled and more deterministic sampling.

## Setup in a Dispatcher checkout

```bash
git clone https://github.com/SomeSunlight/Llama_Dispatcher.git
cd Llama_Dispatcher

git clone https://github.com/SomeSunlight/Llama_Dispatcher_Laptop_WSL.git \
  instances/Laptop_WSL

uv sync
```

The Dispatcher creates `instances/Laptop_WSL/data/metrics.db` locally when required. Do not copy another instance's database unless historical measurements are intentionally being migrated.

## Usage

With AI Workstation, configure the instance and ensemble once and let AI Workstation supply the concrete llama.cpp build and model root.

A direct Dispatcher invocation is still possible:

```bash
uv run src/dispatcher.py serve \
  --ensemble thinkpad-sycl \
  --instance Laptop_WSL \
  --bin-dir /path/to/llama.cpp/build/bin \
  --model-root /path/to/models
```

The `machine_guid` in `instance.yaml` identifies this WSL measurement history. Do not reuse it for a distinct Windows instance.

## License

MIT.
