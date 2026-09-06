# Blackwell (RTX 5090, sm_120) port — setup notes

Target box: `chungyili-Z890`, Ubuntu 24.04.4, RTX 5090 (compute capability **12.0**,
32 GB VRAM), driver 580.173.02 (supports CUDA 13.0), **30 GB RAM / 8 GB swap**,
gcc 13.3, **no passwordless sudo**.

## Why a port is needed

The upstream repo targets **Ubuntu 22.04 + CUDA 12.1 + torch 2.3.1**:

| Where | Pin |
|---|---|
| `pyproject.toml` | `torch = "2.3.1"`, `torchvision = "0.18.1"` |
| `Frosting/setup.bash` | `pytorch3d ... py310_cu121_pyt231` prebuilt wheel |
| `BundleSDF/setup.bash` | header comment: *"written for Ubuntu 22.04 with CUDA 12.1"* |
| README nerfstudio step | `--index-url .../whl/cu121`, `pip==23.0.1` for tiny-cuda-nn |

torch 2.3.1 ships **no sm_120 kernels** (Blackwell support landed in torch 2.7 / cu128),
so every GPU component fails on this box until re-pinned and rebuilt.

## Constraints that shaped the approach

1. **No passwordless sudo** → cannot `apt-get install`. Upstream `BundleSDF/setup.bash`
   opens with a long `sudo apt-get install` list; it must be adapted.
2. **Ubuntu 24.04 ships only CUDA 12.0** (`nvidia-cuda-toolkit` 12.0.140) — too old for
   Blackwell regardless of sudo. NVIDIA's own repo would need root too.
   → **conda-forge / nvidia channel via micromamba**, which needs no root.
3. **30 GB RAM.** Source builds (pytorch3d, tiny-cuda-nn, BundleSDF CUDA) are memory
   hungry and have previously OOM-killed the whole tmux scope including `claude`.
   → every heavy build runs inside its own systemd scope:
   `systemd-run --user --scope -p MemoryMax=12G -p MemorySwapMax=4G -- <cmd>`
   and with a bounded `MAX_JOBS` / `-j` so parallel nvcc does not balloon.

## Version decisions

- **torch 2.7.1+cu128 / torchvision 0.22.1+cu128.** First torch line with real sm_120
  support, and the *closest* to the repo's 2.3.1 — newer lines (up to 2.11+cu128 /
  2.13+cu129 are available for cp310) increase API-drift risk in code written for 2.3.1.
- **CUDA toolkit 12.8** to match the cu128 wheels (extension builds link against torch).
- **Python 3.10** as upstream requires (`>=3.10,<3.11`); installed via `uv python install 3.10`.
- **tiny-cuda-nn**: upstream has a **`cc-120` branch** (compute capability 12.0) and a
  `cuda-13` branch — use `cc-120` rather than patching `master`.
- **pytorch3d**: no prebuilt wheel exists for `py310_cu128_pyt271` (the fbaipublicfiles
  path 404s), so it must be **built from source**. Heaviest single build in the project.

## Stages

- [x] 0. Survey box; clone all 5 submodules (SSH via `gh` account `Chung-I`)
- [x] 1. CUDA 12.8 toolkit + colmap via micromamba (no root) — **done**, see below
- [~] 2. Core Poetry env, torch re-pinned to 2.7.1+cu128 — lock resolved, install running
- [ ] 3. nerfstudio env + tiny-cuda-nn (`cc-120` branch)
- [ ] 4. Frosting: pytorch3d from source, diff-gaussian-rasterization, simple-knn, nvdiffrast
- [ ] 5. Neuralangelo (reuses tiny-cuda-nn)
- [ ] 6. BundleSDF: adapt `setup.bash` to a sudo-free conda-forge dependency set
- [ ] 7. LoFTR `outdoor_ds.ckpt` weights (manual Google Drive download)
- [ ] 8. Benchmark dataset from HuggingFace; end-to-end asset-generation smoke test

## Environments (intentionally separate, as upstream expects)

| Path | Purpose |
|---|---|
| `~/micromamba/envs/r2s-cuda` | nvcc / CUDA 12.8 toolkit (build-time only) |
| `~/micromamba/envs/r2s-colmap` | colmap binary |
| `.venv` | core Poetry env (SAM2 segmentation, asset generation, robot_payload_id) |
| `.venv_nerfstudio` | nerfstudio + tiny-cuda-nn |
| `scalable_real2sim/Frosting/.venv` | Frosting / Gaussian Splatting |
| `scalable_real2sim/neuralangelo/.venv` | Neuralangelo |
| `scalable_real2sim/BundleSDF/.venv` | BundleSDF tracking |

## Notes / gotchas

- Upstream warns of a **segfault with numpy >= 2.0.0** in a data-collection dependency,
  yet `pyproject.toml` pins `numpy = "^2.2.0"`. Only bites `run_data_collection.py`
  (robot-specific, not runnable here anyway).
- `robot_payload_id` is **CPU-only** (Drake, nlopt, nevergrad) and unaffected by the whole
  CUDA question — it installs cleanly and is the half that matters for the mass/CoM work.
  (Its drake ceiling was relaxed during the port; see the Stage 2 log below.)
- Step 3 of the pipeline (robot identification) wants a **MOSEK license** for the SDP.
  Drake can fall back to other SDP solvers (SCS/CSDP) without one; expect slower and
  possibly looser solutions.

---

## Progress log

### Stage 1 — CUDA toolkit + colmap (done, no root)

- `~/micromamba/envs/r2s-cuda`: **nvcc 12.8.93**, 5.3 GB.
  `nvcc --list-gpu-arch` includes **`compute_120`** → Blackwell codegen confirmed.
- `~/micromamba/envs/r2s-colmap`: **COLMAP 4.2.0 (with CUDA)**, 3.6 GB.
  Run it as `micromamba run -n r2s-colmap colmap ...` (or add the env's `lib/` to
  `LD_LIBRARY_PATH`); the bare binary path will not resolve its libraries.

**Bug hit:** the conda-forge `colmap-4.2.0-cuda_130*` build links
`libOpenImageIO.so.3.1` / `libOpenImageIO_Util.so.3.1` but **does not declare
`openimageio` in its run dependencies** (67 deps, none of them openimageio). The binary
therefore installs "successfully" and then dies with
`error while loading shared libraries: libOpenImageIO.so.3.1`.
Fix: `micromamba install -n r2s-colmap -c conda-forge "openimageio>=3.1,<3.2"`.
After that `ldd` reports 0 missing libraries. Upstream packaging bug, not ours.

### Stage 2 — core env: the drake/manipulation knot

`poetry lock` first failed with an opaque internal crash:

```
OverrideNeededError ({Package('opencv-python','4.11.0.86'): {'numpy': >=1.21.4}},
                    {Package('opencv-python','4.11.0.86'): {'numpy': >=1.21.2}})
  ...then: AssertionError at poetry/mixology/partial_solution.py:160
```

Hypotheses tested and **disproved**:
1. Poetry version — same crash on 2.4.3 and 2.1.3 (lock-version 2.1 says upstream used 2.x).
2. Poetry's own interpreter — same crash under Python 3.14.5 and 3.12.3.
3. My torch re-pin — same crash with upstream's original `torch = "2.3.1"` restored.

Root cause found by running the identical constraint set through **uv**, which reported
the real conflict instead of crashing:

```
manipulation==2025.2.16 depends on drake>=0.0.20250131,<0.1   (nightly-only)
and you require drake==1.36.0  ->  unsatisfiable
```

The unsatisfiable constraint forced Poetry deep into override-branch exploration, where it
hit its internal assertion bug. Poetry reported the *symptom* (opencv/numpy override);
uv reported the *cause*. **Lesson: when Poetry crashes internally, re-run the same
constraints through uv to get a readable error.**

Why upstream's pins are unreachable at all:
- old drake nightlies are **pruned** from the index (`0.0.20250425` is gone), and
- current nightlies ship **cp312/cp313/cp314 only — no cp310**, which this project requires.

Resolution (all satisfiable on cp310):
- drake stable ships cp310 wheels **only up to 1.51.1**
- `manipulation >= 2025.5.18` requires `drake >= 1.41.0`
- → **drake 1.41.0 + manipulation 2025.5.18**, the pairing closest to upstream's Feb-2025
  nightly, minimising API drift.
- `robot_payload_id`'s `drake = "<=1.36.0"` had to be relaxed to `>=1.41.0,<1.52.0`.
  Its comment blamed a **1.37.0 segfault** — watch for that when running identification;
  if it reappears, that pin is the first thing to revisit.

Resulting lock: drake 1.41.0, manipulation 2025.5.18, torch 2.7.1+cu128,
torchvision 0.22.1+cu128, numpy 2.2.5, opencv-python 4.11.0.86, sam2 0.4.1, open3d 0.19.0.

### Memory discipline

`poetry install` and every later source build runs inside its own scope:
`systemd-run --user --scope -p MemoryMax=12G -p MemorySwapMax=4G -- <cmd>`
(verified working on this box). Without it an OOM kill takes the whole tmux scope,
including the agent session.
