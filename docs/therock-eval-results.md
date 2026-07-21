# Vulkan vs TheRock ROCm eval — gfx1150

Evaluating whether to vendor an up-to-date TheRock ROCm build for llama.cpp on
this host, against the current Vulkan backend. This doc locks the **Vulkan
baseline ("A" column)** first; a TheRock ROCm "B" column gets added
alongside it once that build exists.

**Result: Vulkan wins on both models, both metrics.** TheRock ROCm 7.13.0 is
3-5% slower on prompt processing and 10-11% slower on token generation than
the Vulkan (RADV) backend on this gfx1150 host. See "TheRock ROCm build" and
"Results" below for the full numbers and the B3 build-effort verdict.

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

## TheRock ROCm build

- SDK: TheRock prebuilt `gfx1150-7.13.0` (ROCm **7.13.0**, `hipconfig --version` →
  `7.13.99004-3309c6114a`), downloaded via lemonade's TheRock fetch path.
  Fetch speed from the S3 bucket: **~350 KB/s** — a 16 GB SDK takes on the
  order of hours to pull cold. This is a real maintenance/CI cost to note if
  vendoring TheRock: re-fetching on cache miss is slow regardless of build
  effort.
- llama.cpp source: same nixpkgs-pinned source as the Vulkan build
  (`nix eval --raw .#llama-cpp-vulkan.src`, using the flake's own pinned
  nixpkgs — the system's default nixpkgs channel resolves to a *different*,
  newer llama.cpp (9925) and must not be used for this comparison). The
  source's `COMMIT` file reads `70b54e1`, confirming it's the exact same
  commit as the Vulkan baseline's `build: 70b54e1 (9608)`. Because the
  copied source tree has no `.git`, the compiled binary's own build banner
  prints `build: unknown (0)` instead of `9608` — the `COMMIT` file is the
  authoritative confirmation of version match, not the runtime banner.
- Build command matched build 9608 exactly in source; only the backend
  (`GGML_HIP=ON`, `AMDGPU_TARGETS=gfx1150`) differs from the Vulkan build.

### B3 build-effort verdict: **hard, but tractable in one focused session (~2 hours)**

Getting `llama-bench` to link and run against the SDK's `amdclang++` required
four non-obvious fixes, none of which are documented anywhere in TheRock's
or llama.cpp's build docs for a NixOS host:

1. **`libatomic.so.1` missing** (known from a prior investigation — see repo
   notes): TheRock's runtime libs need it and NixOS doesn't ship it by
   default. Pulled from `nixpkgs#gcc.cc.lib` and put on `LD_LIBRARY_PATH`.
2. **`amdclang++` can't find a C++ standard library on NixOS** — there's no
   `/usr/include` or `/usr/lib` for it to fall back to. Fixed with
   `--gcc-toolchain=<nixpkgs gcc unwrapped store path>` (finds `cstdlib`/
   `cmath` from GCC's bundled C++ headers) plus `-idirafter <glibc.dev>/include`
   (supplies `math.h` etc. — must be `-idirafter`, not `-isystem`, because
   `#include_next` in GCC's `<cmath>` needs the glibc headers to sit *after*
   GCC's own C++ headers in the search list, and `-isystem` inserts too early).
3. **Linking fails**: `ld.lld` (bundled with the SDK) can't find `Scrt1.o`,
   `crti.o`/`crtn.o`, `-lstdc++`, `-lgcc_s`, `-lc` because none of nixpkgs'
   gcc/glibc paths are on its default search path. Fixed with
   `-B <glibc>/lib -L <glibc>/lib -L <gcc.cc.lib>/lib -Wl,-dynamic-linker,<glibc>/lib/ld-linux-x86-64.so.2`.
4. **The real trap**: CMake's own HIP-compiler-identification probe (which
   determines `CMAKE_HIP_COMPILER_ID`) does its own private compile+link
   using only `CMAKE_HIP_FLAGS` — it does **not** pick up
   `CMAKE_EXE_LINKER_FLAGS`. With the linker flags only set on
   `CMAKE_EXE_LINKER_FLAGS`, the identification probe's link step failed
   silently, `CMAKE_HIP_COMPILER_ID` came back empty, and CMake then
   generated build rules that omitted `-x hip` on `.cu` files (ggml marks
   its CUDA-shaped `.cu` sources as `LANGUAGE HIP` via
   `set_source_files_properties`, but relies on Clang-HIP CMake module
   detection to add `-x hip`). The symptom was a confusing, unrelated-looking
   error: `clang++: error: unsupported CUDA gpu architecture: gfx1150`
   (clang silently fell back to CUDA-mode compilation without `-x hip`, then
   rejected `gfx1150` as an invalid `sm_XX` CUDA arch). Fix: fold the linker
   flags into `CMAKE_HIP_FLAGS` itself, not just `CMAKE_EXE_LINKER_FLAGS`, so
   the identification probe's link step succeeds too.

Once those four fixes were in place, the actual `llama-bench` HIP target
built cleanly with `ninja` in one pass (~378 targets, few minutes). The
final working `cmake` configure line:

```bash
cmake -S <src> -B build -G Ninja \
  -DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1150 -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_HIP_COMPILER=<SDK>/bin/amdclang++ \
  -DCMAKE_PREFIX_PATH=<SDK> \
  -DCMAKE_HIP_FLAGS="--rocm-path=<SDK> --gcc-toolchain=<gcc-unwrapped> -idirafter <glibc.dev>/include -B <glibc>/lib -L <glibc>/lib -L <gcc.cc.lib>/lib -Wl,-dynamic-linker,<glibc>/lib/ld-linux-x86-64.so.2" \
  -DCMAKE_EXE_LINKER_FLAGS="-B <glibc>/lib -L <glibc>/lib -L <gcc.cc.lib>/lib -Wl,-dynamic-linker,<glibc>/lib/ld-linux-x86-64.so.2" \
  -DCMAKE_SHARED_LINKER_FLAGS="<same as above>" \
  -DCMAKE_MODULE_LINKER_FLAGS="<same as above>" \
  -DLLAMA_CURL=OFF -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_EXAMPLES=ON
```

None of this is exotic once found, but every one of the four failures
produced a misleading or generic error message (`cmath` not found,
`unable to find library -lstdc++`, and — worst — the `unsupported CUDA gpu
architecture` red herring), so getting from "TheRock SDK on disk" to "GPU
executing HIP kernels" took real debugging, not just following a recipe.
Vendoring this into the flake would mean baking all of the above into a
Nix derivation (a `--gcc-toolchain`/glibc-path-aware wrapper around
`amdclang++`, essentially reimplementing part of what nixpkgs' own
`rocmPackages` wrapper already does) — moderate ongoing maintenance cost,
not a one-off patch.

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
| gemma4 26B.A4B Q4_K - Medium | 25.23 B | **Vulkan (baseline)** | 326.38 ± 1.58 | 19.67 ± 0.09 | 310.42 ± 6.31 (**-4.9%**) | 17.57 ± 0.34 (**-10.7%**) |
| qwen3 8B Q8_0 | 8.19 B | **Vulkan (baseline)** | 461.13 ± 5.21 | 9.46 ± 0.01 | 445.80 ± 7.12 (**-3.3%**) | 8.51 ± 0.11 (**-10.0%**) |

**Vulkan (RADV) beats TheRock ROCm 7.13.0 on every metric, both models.**
The gap is small on prompt processing (3-5% slower) but consistent and
larger on token generation (10-11% slower). Percentages are TheRock relative
to the Vulkan baseline (negative = TheRock slower).

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

**gemma-4-26B-A4B-it (large, MoE) — TheRock ROCm 7.13.0:**

```
ggml_cuda_init: found 1 ROCm devices (Total VRAM: 27935 MiB):
  Device 0: AMD Radeon 890M Graphics, gfx1150 (0x1150), VMM: no, Wave Size: 32, VRAM: 27935 MiB
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| gemma4 26B.A4B Q4_K - Medium   |  15.70 GiB |    25.23 B | ROCm       | 999 |           pp256 |        310.42 ± 6.31 |
| gemma4 26B.A4B Q4_K - Medium   |  15.70 GiB |    25.23 B | ROCm       | 999 |           tg128 |         17.57 ± 0.34 |

build: unknown (0)
```

(`build: unknown (0)` — see "TheRock ROCm build" above: the copied source
tree has no `.git`, so `llama.cpp`'s build-info script can't populate the
banner. Source `COMMIT` file confirms `70b54e1`, matching the Vulkan
baseline exactly.)

**Qwen3-8B (small, dense) — TheRock ROCm 7.13.0:**

```
ggml_cuda_init: found 1 ROCm devices (Total VRAM: 27935 MiB):
  Device 0: AMD Radeon 890M Graphics, gfx1150 (0x1150), VMM: no, Wave Size: 32, VRAM: 27935 MiB
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| qwen3 8B Q8_0                  |   8.11 GiB |     8.19 B | ROCm       | 999 |           pp256 |        445.80 ± 7.12 |
| qwen3 8B Q8_0                  |   8.11 GiB |     8.19 B | ROCm       | 999 |           tg128 |          8.51 ± 0.11 |

build: unknown (0)
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

The TheRock ROCm runs were monitored the same way (`amdgpu_top -J -n <N> -s
150` for the process lifetime).

**Qwen3-8B, TheRock ROCm** (183 samples across the whole run):

- max GFX busy: 100%
- mean GFX busy: 84.8%
- samples >50% busy: 157 / 183
- last 20 samples (steady state): `100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100`

**gemma-4-26B-A4B, TheRock ROCm** (250 samples; monitor window ran longer
than the bench process, so the tail is post-exit idle, not part of the run):

- max GFX busy: 100%
- mean GFX busy: 63.6% (skewed down by ~150 idle samples after the process exited)
- samples >50% busy: 157 / 250
- first 20 samples (idle during load): all `0`
- samples 100-120 (steady state, mid-run): `100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100, 100`

`llama-bench`'s startup banner also confirms the ROCm device was selected
(`ggml_cuda_init: found 1 ROCm devices ... Device 0: AMD Radeon 890M
Graphics, gfx1150 (0x1150)`, `backend | ROCm` and `ngl 999` in the results
table) — no CPU fallback occurred on either run.

## Status

**Done.** Both baselines measured with the same methodology, same model
files, same matched llama.cpp source commit (`70b54e1` / nixpkgs build
9608). Vulkan (RADV) outperforms TheRock ROCm 7.13.0 on this gfx1150 host
across both models and both metrics — see the summary at the top of this
doc and the B3 build-effort section for why TheRock is still notably harder
to stand up than Vulkan (which just works via nixpkgs' `llama-cpp-vulkan`
with zero extra toolchain wrangling).
