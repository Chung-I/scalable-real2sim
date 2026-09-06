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

### Gotcha — Poetry keyring / DBus on this box

`poetry install` died mid-run with:

```
DBusErrorResponse [org.freedesktop.DBus.Error.UnknownMethod]
  ('Object does not exist at path "/org/freedesktop/secrets/collection/_1b_96_ff_bf_ebV"')
  -> ItemNotFoundException: Item does not exist!
Cannot install msgpack.
```

Poetry probes the SecretStorage keyring (`poetry lock -v` even prints
"Checking keyring availability: Available"), but this session has no unlocked
secret-service collection, so the lookup raises and the install aborts partway.
Nothing to do with dependencies — it stopped on `msgpack`, right after the nvidia
CUDA wheels.

Fix (both applied, belt and braces):
```bash
poetry config keyring.enabled false
export PYTHON_KEYRING_BACKEND=keyring.backends.null.Keyring
```
`poetry install` is resumable — re-running skips what already landed.

### Stage 6 plan — BundleSDF without root

`BundleSDF/setup.bash` (274 lines) is the hardest piece. As written it:

1. `sudo apt-get install`s ~50 `-dev` packages (gtk, qt5, ffmpeg codecs, boost, flann,
   glog, gflags, yaml-cpp, zeromq, hdf5, proj, protobuf, ...) — **needs root, unavailable**;
2. builds **Eigen 3.4.0, OpenCV + opencv_contrib, PCL, and pybind11 from source**, each
   with `make -j$(nproc)` — on 30 GB RAM an unbounded OpenCV/PCL build is a near-certain
   OOM, which on this box takes the whole session with it;
3. pins a stack that cannot work on Blackwell:
   `torch==2.4.0` (no sm_120) and
   `pytorch3d ... py310_cu118_pyt201` (**CUDA 11.8**), plus `numpy==1.26.1`,
   `Cython==0.29.20`, `scikit-image==0.17.2`, `networkx==2.2`.

Planned adaptation (do **not** run `setup.bash` as-is):

- Replace the whole apt list *and* the four source builds with a dedicated micromamba
  env (`r2s-bundlesdf`) pulling `eigen`, `opencv`, `pcl`, `boost-cpp`, `flann`, `glog`,
  `gflags`, `yaml-cpp`, `zeromq`, `hdf5`, `proj`, `protobuf`, `pybind11` from
  conda-forge. This removes hours of compiling and the main OOM risk, and needs no root.
- Point BundleSDF's own CMake at that env via `CMAKE_PREFIX_PATH`.
- Bump `torch==2.4.0` -> `2.7.1+cu128`; build **pytorch3d from source** (shared problem
  with Stage 4, so build the wheel once and reuse it in both venvs).
- The ancient pins (`scikit-image==0.17.2`, `networkx==2.2`, `Cython==0.29.20`) will
  likely refuse to build against modern setuptools on cp310 — expect to relax them.
- Any remaining `make` must be bounded (`-j4`), never `-j$(nproc)`, and run inside the
  systemd scope.

Note BundleSDF keeps its own venv with `numpy==1.26.1`, which conflicts with the core
env's numpy 2.2.5 — that is fine and intentional, they are separate environments.

### Stage 2 — Blackwell validation (the milestone that matters)

With `.venv` built from the re-pinned lock:

```
torch      : 2.7.1+cu128
cuda build : 12.8
available  : True
device     : NVIDIA GeForce RTX 5090
capability : (12, 0)
arch list  : ['sm_75','sm_80','sm_86','sm_90','sm_100','sm_120','compute_120']
matmul     : OK   (4096x4096 on device)
```

**`sm_120` is in the arch list and a real matmul executes on the GPU.** This is the
assumption the entire port rests on — upstream's torch 2.3.1 stops at sm_90 and would
fail here with "no kernel image is available for execution on the device". Everything
downstream (tiny-cuda-nn, pytorch3d, the rasterizers, BundleSDF) can now be built against
a torch that actually targets this GPU.

Also of note: `poetry install` had to be run **twice** — see the keyring gotcha above.
The first run got as far as the nvidia CUDA wheels before aborting.

### Stage 2 — post-install verification

A clean install does not prove the 18-month jump from upstream's Feb-2025 drake nightly
to drake 1.41.0 is harmless, so the affected imports were exercised explicitly:

| Import | Result |
|---|---|
| `pydrake` | OK (no segfault — the old 1.37.0 worry did not materialise) |
| `manipulation.utils.ConfigureParser` | OK |
| `manipulation.station` → `MakeHardwareStation`, `Scenario`, `LoadScenario` | OK |
| `sam2`, `open3d` 0.19.0, `trimesh`/`coacd`/`vhacdx` | OK |
| `robot_payload_id` + `.optimization` `.symbolic` `.data` `.utils` `.environment` `.control` | **all OK** |

`optimization` and `symbolic` are the modules most exposed to Drake API drift (they drive
MathematicalProgram, the SDP solve, and symbolic regressor construction), so those passing
is the real evidence that drake 1.41.0 + manipulation 2025.5.18 is a sound pairing.

**SDP solvers** (the README says robot identification wants a MOSEK license):

```
Mosek     available=True   enabled=False   <- bundled, but no license
SCS       available=True   enabled=True
CSDP      available=True   enabled=True
Clarabel  available=True   enabled=True
```

So identification will run, falling back to SCS/CSDP/Clarabel for the pseudo-inertia
constraint (J ≻ 0). Expect slower solves and possibly looser solutions than the paper.
To use MOSEK, drop a licence at `~/mosek/mosek.lic` (free for academics) — `enabled`
flips to True with no code change.

### Gotcha — Poetry's bundled git client (dulwich)

The pinned `transformers` git revision failed to install with:

```
GitProtocolError: Length of pkt read 1532 does not match length prefix 4005
  at dulwich/protocol.py:270 in read_pkt_line
Cannot install transformers.
```

Poetry's pure-Python git client (dulwich) mishandles the protocol on a repo the size of
huggingface/transformers. Fix: `poetry config system-git-client true` to use the real
`git` binary. Everything else in the lock had already installed by this point.

### Stage 2 complete

`poetry install` EXIT=0 on the third attempt (resolver fix, then keyring, then dulwich).
`.venv` = 9.2 GB. `transformers 4.49.0.dev0` at the pinned rev, `AutoModel` (the DINO path
used by segmentation) imports, `sam2` imports, `run_asset_generation.py` parses.
`poetry check` emits only cosmetic warnings about upstream's `[tool.poetry]` style.

### Stage 3 — native CUDA toolchain verified before building anything

Before attempting tiny-cuda-nn, confirmed the full native path works, since every
remaining stage (tcnn, pytorch3d, the GS rasterizers, BundleSDF) depends on it:

```bash
nvcc -arch=sm_120 -o /tmp/t /tmp/t.cu   # CUDA 12.8 + host gcc 13.3  -> COMPILE OK
/tmp/t                                   # -> "hi"   (ran on the 5090)
```

So CUDA 12.8's nvcc accepts Ubuntu 24.04's gcc 13.3 as host compiler *and* emits working
sm_120 code. No need for a conda `gxx_linux-64` shim.

Build environment for every CUDA extension from here on:
```bash
export CUDA_HOME=$HOME/micromamba/envs/r2s-cuda
export PATH=$CUDA_HOME/bin:$PATH
export TCNN_CUDA_ARCHITECTURES=120
export TORCH_CUDA_ARCH_LIST="12.0"
export MAX_JOBS=4          # never $(nproc): 30 GB RAM, parallel nvcc balloons
```
