# Vulkan vs TheRock ROCm eval — gfx1150

Evaluating whether to vendor an up-to-date TheRock ROCm build for llama.cpp on
this host, against the current Vulkan backend. This doc locks the **Vulkan
baseline ("A" column)** first; a TheRock ROCm "B" column gets added
alongside it once that build exists. Task 1 of the eval — do not add ROCm
numbers here until they've been measured with the same methodology.

## Host

- Arch: **gfx1150** (AMD Radeon 890M, Strix Point APU)
- Kernel: `uname -r` → `7.1.3`
- RAM: 54 GiB total (UMA, no dedicated VRAM)
- `rocminfo | grep 'Name: *gfx'` → `gfx1150`

## Build under test

- Package: `.#llama-cpp-vulkan` (built via `nix build --no-link --print-out-paths .#llama-cpp-vulkan`)
- Store path: `/nix/store/mqq37s52wv4z59wa4g5gccij5iacjawp-llama-cpp-9608`
- llama.cpp build (from `llama-bench` banner): `build: 70b54e1 (9608)`
- Vulkan device reported by llama-bench: `AMD Radeon 890M Graphics (RADV STRIX1) (radv) | uma: 1 | fp16: dot2 | bf16: 0 | warp size: 64 | shared memory: 65536 | int dot: 1 | matrix cores: KHR_coopmat`

## Methodology

Standard bench matching the README's existing numbers (prompt 256 / gen 128, 3
reps after warmup, `-ngl 999` to force full GPU offload):

```bash
<out>/bin/llama-bench -m "<MODEL>" -ngl 999 -p 256 -n 128 -r 3 -o md
```

Two models, picked to bracket size/architecture:

| Size  | Model | Exact path | Quant |
| ----- | ----- | ---------- | ----- |
| Large (MoE) | `unsloth/gemma-4-26B-A4B-it-GGUF` (README's model) | `/home/noams/.cache/huggingface/hub/models--unsloth--gemma-4-26B-A4B-it-GGUF/snapshots/8bacec5c8e829a25502cdfe3c3f5b6aabee3218c/gemma-4-26B-A4B-it-UD-Q4_K_M.gguf` | UD-Q4_K_M |
| Small (dense) | `unsloth/Qwen3-8B-GGUF` | `/home/noams/.cache/huggingface/hub/models--unsloth--Qwen3-8B-GGUF/snapshots/a6adef130ffb23ddaf1a62fec9dced968c9bc482/Qwen3-8B-Q8_0.gguf` | Q8_0 |

## Results

| Model | Params | Backend | pp256 t/s | tg128 t/s | TheRock ROCm pp256 t/s | TheRock ROCm tg128 t/s |
| ----- | ------ | ------- | --------: | --------: | ---------------------: | ---------------------: |
| gemma4 26B.A4B Q4_K - Medium | 25.23 B | **Vulkan (baseline)** | 326.38 ± 1.58 | 19.67 ± 0.09 | TBD | TBD |
| qwen3 8B Q8_0 | 8.19 B | **Vulkan (baseline)** | 461.13 ± 5.21 | 9.46 ± 0.01 | TBD | TBD |

Full raw `llama-bench -o md` output per model:

**gemma-4-26B-A4B-it (large, MoE):**

```
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| gemma4 26B.A4B Q4_K - Medium   |  15.70 GiB |    25.23 B | Vulkan     | 999 |           pp256 |        326.38 ± 1.58 |
| gemma4 26B.A4B Q4_K - Medium   |  15.70 GiB |    25.23 B | Vulkan     | 999 |           tg128 |         19.67 ± 0.09 |

build: 70b54e1 (9608)
```

**Qwen3-8B (small, dense):**

```
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| qwen3 8B Q8_0                  |   8.11 GiB |     8.19 B | Vulkan     | 999 |           pp256 |        461.13 ± 5.21 |
| qwen3 8B Q8_0                  |   8.11 GiB |     8.19 B | Vulkan     | 999 |           tg128 |          9.46 ± 0.01 |

build: 70b54e1 (9608)
```

## Validity gate — GPU utilisation evidence

Both runs were monitored with `amdgpu_top -J -n 1 -s 150` sampled continuously
for the process lifetime (JSON `devices[0].gpu_activity.GFX` field = GRBM GFX
busy %). Idle baseline before the model started computing was ~1-3%; both
runs climbed to ~99-100% busy during the pp/tg phases, confirming the Vulkan
backend actually executed on the GPU rather than falling back to CPU.

**Qwen3-8B** (213 samples across the whole run):

- max GFX busy: 100%
- mean GFX busy: 94.8%
- samples >50% busy: 205 / 213
- first 20 samples (ramp-up from idle): `1, 2, 7, 12, 17, 17, 31, 43, 53, 63, 70, 75, 80, 84, 87, 89, 91, 92, 93, 95`
- last 20 samples (steady state): `100, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 86`

**gemma-4-26B-A4B** (163 samples across the whole run):

- max GFX busy: 99%
- mean GFX busy: 69.5% (lower mean because model load + first pp warmup dominate more of the shorter overall wall time before ramping up)
- samples >50% busy: 109 / 163
- first 20 samples (idle during load): `2, 2, 3, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 3, 4, 3, 3, 3`
- last 20 samples (steady state): `99, 99, 99, 99, 99, 99, 98, 99, 99, 99, 99, 99, 99, 99, 99, 99, 99, 98, 96, 80`

`llama-bench`'s own startup banner also confirms the Vulkan device was selected
and layers offloaded (`ngl 999` in the results table, `ggml_vulkan: Found 1
Vulkan devices` / `load_backend: loaded Vulkan backend` in stderr) — no CPU
fallback occurred.

## Status

Vulkan baseline locked. Next step: build/vendor an up-to-date TheRock ROCm
llama.cpp, run the identical commands against the identical model paths, and
fill in the "TheRock ROCm" columns above.
