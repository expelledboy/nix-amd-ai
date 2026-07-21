# TheRock ROCm integration — design

**Status:** approved design, pre-plan
**Date:** 2026-07-21
**Goal:** latest ROCm for the best experience on Strix Halo (gfx1151), without wrecking the low-maintenance auto-bump workflow the flake relies on today.

## Problem

The flake builds its ROCm backends against nixpkgs' pinned ROCm (currently 7.2.3). That ROCm has known **gfx1151 (Strix Halo)** runtime faults — memory-access faults on VRAM access (see nixpkgs #472876, ROCm #5853/#6034). AMD's newer ROCm ships via **TheRock**, whose nightlies carry native gfx1151 kernels. We want that newer ROCm for the backends, on both target GPUs: **gfx1150** (current P14s daily driver) and **gfx1151** (incoming Strix Halo).

## Hard constraint (verified)

TheRock is a **monolithic prebuilt SDK tarball, per gfx arch** (e.g. `therock-dist-linux-gfx1151-7.11.0aNNNN.tar.gz`), from a fast-moving nightly S3 bucket. It is **not** shaped like nixpkgs `rocmPackages` (no separate `clr`/`hipblas`/`rocblas` outputs). Consequences:

1. **No drop-in `rocmPackages` override.** `pkgs.llama-cpp-rocm` / `stable-diffusion-cpp.override { rocmSupport = true; }` cannot pick up TheRock. Each backend must be **rebuilt against the SDK** with custom cmake (+ likely `hip-version-fix` / `rocwmma-compat` patches).
2. **Per-arch builds.** gfx1150 and gfx1151 are separate SDK bundles → separate backend builds.
3. Packaging the SDK itself is easy: `fetchurl` + `autoPatchelfHook`. This also *solves* the `libatomic.so.1` breakage from #57 — Nix supplies the deps the raw lemonade download couldn't.

Community precedent confirms this exact pattern: `demyanrogozhin/nix-llama-rocm` (`pkgs/rocm7-bin.nix`, `rocm-sources.json`, `update-rocm.py`) and `hellas-ai/nix-strix-halo` (`overlays/rocm.nix` `therock-bin` provider). Both **rebuild backends against the SDK**; neither yields a drop-in `rocmPackages`.

## Core principle: the auto-merge safety net has a hole

Today auto-merge is safe because a **green build ⇒ a nixpkgs-vetted, known-working ROCm** — nixpkgs absorbs the "does it actually run" work. Building against TheRock **breaks that proxy**: nightlies compile clean and then fault on hardware (exactly the bug class we're chasing), and CI has **no gfx1151 GPU** to catch it. Two moving upstreams (ROCm + backend HEAD) also multiply patch-rot.

Therefore "latest ROCm" structurally fights "low-touch auto-merge." We do **not** try to make the TheRock path auto-merge. We **quarantine** it so it can't contaminate the lane that already works.

## Chosen approach

**Vendor TheRock in-repo, deliberately pinned, phased llama.cpp-first, in a separate hand-gated update lane, with the nixpkgs ROCm output kept as a live fallback.**

Rejected: consuming `hellas-ai` as a flake input — it couples our reproducibility + Cachix to a third-party flake's API/cadence, and since they likely don't ship sd-cpp/whisper we'd build those against their SDK anyway (hard part of vendoring + coupling risk). Use it as a **cookbook** for cmake flags/patches, not a dependency.

## Architecture

### 1. TheRock SDK package (`pkgs/rocm-therock/`)
- `default.nix`: `fetchurl` the pinned tarball + `autoPatchelfHook` → monolithic SDK at `$out` (bin/, lib/). Parameterised by gfx target. Model on `demyanrogozhin/pkgs/rocm7-bin.nix`.
- `rocm-sources.json`: `{ url, sha256, version }` **per arch** (`gfx1150`, `gfx1151`). The pin.
- One SDK derivation instance per target.

### 2. Backends rebuilt against the SDK
- **Phase 1: llama.cpp only**, both arches. Custom derivation (or a targeted override) with cmake pointed at the SDK + any required patches. Cribbed from the community flakes.
- **Phase 2 (later): stable-diffusion.cpp, whisper.cpp** against the SDK, once the phase-1 loop is proven on hardware.
- Backends not yet on TheRock stay on nixpkgs untouched.

### 3. Per-arch selection & fallback
- Expose TheRock-built backends per arch (e.g. `llama-cpp-therock-gfx1150`, `-gfx1151`).
- A module option selects the arch the host wires into lemonade (default: explicit; auto-detect is an open question). The `/etc/lemonade/backends/*` symlink wiring is unchanged.
- **Keep the existing nixpkgs `llama-cpp-rocm` output alive as a named fallback**, so a bad SDK bump degrades to "old ROCm still boots," not "daily driver bricked." Possibly a module toggle to choose provider (`nixpkgs` vs `therock`).

### 4. #57 interaction
The `will_install_therock → false` lemonade patch (PR #59) **stays** — it stops lemonade's *runtime* TheRock download. Our vendored TheRock is what the backends link against at build time. No conflict; complementary.

## Update automation (the maintainability core)

- **Deliberate pins, never weekly-auto.** Pin a chosen TheRock build (release/branch tag if TheRock offers one usable — *open question*; else a hand-picked nightly). Bump deliberately, a few times a quarter — not on the nightly firehose. This single choice is what keeps the project maintainable.
- **Split update lanes.** The existing nixpkgs lane (`check-updates.sh` / `bump-versions.sh` / auto-merge) is **untouched**. TheRock gets its own path with `auto-merge = false`, surfaced through the existing "tricky bumps get a hand-bump flag" mechanism — TheRock is *always* the tricky case.
- **Hardware smoke gate.** A TheRock bump PR is gated on a real-GPU smoke test (load a model, generate ~50 tokens, exit 0) on **both** arches — incoming Strix Halo = gfx1151 runner, P14s = gfx1150. A self-hosted runner is ideal; a manual "runtime-OK on both arches" checkbox on the PR is an acceptable solo-maintainer substitute. The build gate proves it *compiles*; only the smoke test proves it *works*.

## Phasing

- **Phase 1:** SDK package + `rocm-sources.json` (both arches) + llama.cpp-on-TheRock (both arches) + fallback + quarantined bump lane + smoke gate. Ship, validate on hardware.
- **Phase 2:** extend to stable-diffusion.cpp + whisper.cpp against the SDK, only if Phase 1's patch-rot cost proves acceptable.

Each phase is its own plan → implementation cycle.

## Testing / validation

- **Build gate:** `nix flake check` + `nix build` the TheRock backends per arch (CI, no GPU).
- **Runtime gate (required for TheRock lane):** hardware smoke test per arch before any TheRock bump merges.
- **Regression:** nixpkgs lane + fallback output continue to build and boot unchanged.

## Open questions (resolve during planning)

1. Does TheRock publish usable **release/branch tags**, or are curated nightlies the only realistic pin? (Determines pin discipline concretely.)
2. Exact **backend build recipe** against the SDK — reuse a nixpkgs `llama-cpp` override with `rocmPackages` faked to the SDK, or a from-scratch derivation? Which patches are actually needed for gfx1150 vs gfx1151?
3. **Per-arch selection UX** — explicit module option vs auto-detect from the running GPU.
4. **Smoke-test runner** — self-hosted GH runner on the Strix Halo box, or manual checkbox to start.
5. gfx1150 SDK bundle availability + naming on the nightly bucket (lemonade already pulled a `gfx1150-7.13.0`, so it exists — confirm the exact tarball name for the pin).

## Out of scope

- Making the TheRock path auto-merge (deliberately rejected).
- vLLM / PyTorch / Python-wheel side of the community flakes.
- Replacing nixpkgs ROCm for anything other than the named backends.

## Related

- #57 / PR #59 (lemonade TheRock runtime-download fix) — complementary.
- Memory: incoming-strix-halo-128gb, upstream-versions-to-watch, perf-baselines-gfx1150, rocwmma-build-flag (rocWMMA was a regression on gfx1150 — re-test per arch before enabling on TheRock).
- Cookbook refs: `demyanrogozhin/nix-llama-rocm`, `hellas-ai/nix-strix-halo`.
