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
- [x] 2. Core Poetry env, torch re-pinned to 2.7.1+cu128 — **done**, sm_120 validated
- [x] 3. nerfstudio env + tiny-cuda-nn (`cc-120` branch) — **done**
- [x] 4. Frosting: pytorch3d 0.7.9 + rasterizers + nvdiffrast — **done**
- [x] 5. Neuralangelo — **done**
- [x] 6. BundleSDF: kaolin + mycuda + BundleTrack — **done**
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

---

## The mass/CoM half is reproducible *without* the CUDA port

Dataset survey (HF `nepfaff/scalable-real2sim`, 205 files, **71 GB** total):

| Path | Size |
|---|---|
| `scalable_real2sim_benchmark_dataset/object_data` (25 `.tar`, e.g. `inflator.tar` 3.6 GB) | 67.5 GB |
| `scalable_real2sim_model_weights/` (SAM2 gripper finetune 2.7 GB + bbox detector 0.8 GB) | 3.5 GB |
| **`scalable_real2sim_benchmark_dataset/robot_system_id_data`** | **0.07 GB** |

**The README's example command is stale:** it points at
`scalable_real2sim_benchmark_dataset_inflator_only/`, which does not exist in the released
dataset. Use `scalable_real2sim_benchmark_dataset/object_data` and
`.../robot_system_id_data` instead.

`robot_system_id_data` is only **70 MB** and needs no GPU, no reconstruction stack, and
none of this port — just the working `.venv`. It contains 5 gripper openings
(0.00–0.10 m) x 10 runs, each with `joint_positions/velocities/accelerations/torques.npy`
at **10,000 samples, 1 kHz, 10 s, 7 joints**, plus `identified_robot_params.npy`.

That params file is a dict of **91 = 13 x 7 links** entries, exactly the paper's
alpha ∈ R^(13N):
`m, hx, hy, hz, Ixx, Ixy, Ixz, Iyy, Iyz, Izz, reflected_inertia, viscous_friction,
dynamic_dry_friction`.

Recomputing `p_com = h/m` from the released parameters:

```
link |   m (kg) |        p_com = h/m (m)       | viscous | dry fric
  0  |   1.6375 | [ 0.0000 -0.0113  0.1130]    |  0.3431 |  0.3594
  1  |   8.4039 | [-0.0020  0.1020  0.0236]    |  0.3092 |  0.4749
  2  |   2.5080 | [-0.0108  0.0167  0.0940]    |  0.0626 |  0.3684
  3  |   3.1837 | [ 0.0245  0.0752  0.0120]    |  0.0414 |  0.4318
  4  |   2.4998 | [ 0.0061  0.0210  0.0593]    |  0.1684 |  0.3406
  5  |   1.1527 | [-0.0087 -0.0381 -0.0130]    |  0.0928 |  0.3522
  6  |   4.1541 | [ 0.0004  0.0000  0.0341]    |  0.0457 |  0.1361
```

**Total identified arm mass 23.540 kg vs the KUKA iiwa 7 datasheet ~23.9 kg — 1.5% error,
from joint torques alone.** Independent corroboration of the paper's ~1.3% mass accuracy
claim, computed here rather than taken on trust. Link-6 mass across the five independent
gripper-opening identifications spans 4.1489–4.1848 kg (0.9%), which is the method's
repeatability.

Note the CoM values are all physically sensible (within ~10 cm of their joint origins),
and that CoM never appears as a free parameter — it is only ever `h/m`, which is why its
error tracks the mass error so closely while the inertia tensor is far worse.

### Stage 3 — tiny-cuda-nn build notes

`uv pip install --no-build-isolation "git+https://github.com/NVlabs/tiny-cuda-nn@cc-120#subdirectory=bindings/torch"`
with `TCNN_CUDA_ARCHITECTURES=120`, `MAX_JOBS=4`, `CUDA_HOME` pointing at the micromamba
CUDA 12.8 env.

- `--no-build-isolation` is **required**: tcnn's `bindings/torch/setup.py` imports `torch`
  at build time, so an isolated build env (which has no torch) fails. Install
  `setuptools wheel ninja` into the target venv first.
- Resolution alone took **4m10s** — the `cc-120` branch drags in large submodules (cutlass).
- The heavy phase is `ptxas -arch sm_120` on `cutlass_mlp.ptx` and `encoding.ptx`;
  individual `ptxas`/`cicc` processes peak around **1.0–1.1 GB RSS each**, so with
  `MAX_JOBS=4` the build sits comfortably under the 12 GB scope cap.
- Even so, this build generates enough system-wide memory pressure to trip the agent
  harness's watchdog, which killed the *polling* processes twice while the build itself
  continued untouched inside its systemd scope. Exactly the outcome the scope is for:
  without it, that pressure is what has previously killed whole sessions on this box.
  Poll the log manually rather than holding a background waiter open during this build.

### Stage 3 — the conda-CUDA layout trap (and the fix)

First tcnn build failed with:

```
fatal error: cuda_runtime_api.h: No such file or directory
ninja: build stopped: subcommand failed.
RuntimeError: Error compiling objects for extension
```

The `.cu` files compiled fine (ptxas ran on `cutlass_mlp.ptx`), and the command line showed
`-DTCNN_MIN_GPU_ARCH=120`, so Blackwell targeting was correct. What failed was the **host
C++** compile.

Cause: conda's `cuda-toolkit` puts headers under
`$CUDA_HOME/targets/x86_64-linux/include/`, while `$CUDA_HOME/include` **already exists as
a different directory** (binutils/OpenCL headers from other conda packages). torch's
`cpp_extension` blindly adds `-I$CUDA_HOME/include`, which therefore contains no CUDA
headers. It also emits `-L$CUDA_HOME/lib64`, and conda has only `lib`.

Fix — make the conda env look like a standard CUDA install:
```bash
ln -sfn $CUDA_HOME/targets/x86_64-linux/include/* $CUDA_HOME/include/
[ -d $CUDA_HOME/lib64 ] || ln -sfn $CUDA_HOME/lib $CUDA_HOME/lib64
```
plus, for good measure, `CPATH=$CUDA_HOME/targets/x86_64-linux/include` and
`LIBRARY_PATH=$CUDA_HOME/lib`.

**This will apply to every remaining CUDA extension** (pytorch3d, diff-gaussian-
rasterization, simple-knn, nvdiffrast, BundleSDF), not just tcnn.

Result — tiny-cuda-nn builds and runs on sm_120:
```
HashGrid + FullyFusedMLP  forward OK -> (4096, 4) torch.float16;  backward OK
```
`FullyFusedMLP` is the most architecture-sensitive kernel in the project, so this is a
strong signal for the remaining builds.

**Logging gotcha:** `echo "EXIT=$?" >> log` appended to the last line because uv's output
has no trailing newline, so `grep -q "^EXIT="` never matched and the waiters hung on an
already-failed build. Use `printf "\nDONE_EXIT=%s\n"`.

### Stage 4 prep — empty vendored glm

`Frosting/gaussian_splatting/submodules/diff-gaussian-rasterization/third_party/glm` ships
**empty** (Frosting has no `.gitmodules`, and the rasterizer/simple-knn sources are vendored
directly, so nothing populates it). The rasterizer only needs `glm/glm.hpp`:
```bash
git clone --depth 1 --branch 0.9.9.8 https://github.com/g-truc/glm third_party/glm
```
The rasterizer's `setup.py` hardcodes no architecture — it only passes the glm include —
so `TORCH_CUDA_ARCH_LIST="12.0"` is what drives sm_120 codegen.

### Stage 3 complete

```
nerfstudio   1.1.5      gsplat     1.4.0
tinycudann   1.7        nerfacc    0.5.2
torch        2.7.1+cu128   cuda: True  NVIDIA GeForce RTX 5090
methods: 46 | nerfacto present: True | gsplat rasterization import OK
```
Only deprecation warnings (`torch.cuda.amp.custom_fwd` → `torch.amp.custom_fwd`), harmless.

**nerfstudio gotcha:** `fpsample==1.0.2` (a nerfstudio dependency) uses `scikit_build_core`
as its build backend but never declares it, which is invisible until you pass
`--no-build-isolation` (required here because tcnn/gsplat import torch at build time).
Fix: `uv pip install scikit_build_core cmake pybind11` into the venv first.

### Stage 4 — Frosting

`requirements.txt` is a 138-line `pip freeze` pinning `torch==2.3.0`, `torchvision==0.18.0`,
`open3d==0.17.0`. Generated `requirements-blackwell.txt` which:
- drops `torch`/`torchvision`/`torchaudio` (installed separately as `+cu128` **first**, so
  the freeze cannot pull a CPU/cu121 build over them — verified afterwards that the venv
  still reports `2.7.1+cu128`),
- bumps `open3d` 0.17.0 → 0.19.0 (0.17 is too old for this stack),
- leaves `numpy==1.26.2` (fine with torch 2.7, and pytorch3d prefers numpy 1.x).

Pre-staged before building:
- `third_party/glm` ← glm 0.9.9.8 (shipped empty, see above)
- `nvdiffrast` ← cloned; it JIT-compiles its CUDA plugin on first use, so it needs
  `CUDA_HOME` at **runtime**, not just build time.

**pytorch3d is built from `main`, not the `V0.7.8` tag.** No prebuilt wheel exists for
`py310_cu128_pyt271` (upstream's `py310_cu121_pyt231` URL is dead for us), and V0.7.8
predates torch 2.7 by a wide margin, so it risks removed ATen APIs. `main` tracks current
torch. Fallback is V0.7.8 if `main` fails.

### Stage 4 complete — Frosting

```
torch 2.7.1+cu128 | pytorch3d 0.7.9 (built from main)
diff_gaussian_rasterization  import OK
simple_knn distCUDA2         CUDA kernel OK -> (1000,)
nvdiffrast                   import OK (plugin JIT-compiles on first rasterize)
```

Two failures, both worth recording:

**1. GCC 13, not Blackwell.** `rasterizer_impl.h` and `simple_knn.cu` use
`uint32_t`/`uint64_t`/`std::uintptr_t` with no `#include <cstdint>`:
```
rasterizer_impl.h(24): error: namespace "std" has no member "uintptr_t"
rasterizer_impl.h(40): error: identifier "uint32_t" is undefined
```
GCC 13 removed the transitive `<cstdint>` includes this 2023-era code relied on. Patched
both files (simple-knn uses fixed-width ints 13x and would have failed immediately after).
**This is an upstream bug that breaks on any modern distro — worth a PR to Frosting.**

**2. simple-knn must NOT be installed editable.** Its `setup.py` declares no `packages=`
and there is no `simple_knn/__init__.py`, so the editable finder maps nothing and
`import simple_knn` fails even though `simple_knn/_C*.so` built fine. Install it
non-editable (`pip install submodules/simple-knn/`). diff-gaussian-rasterization *does*
have an `__init__.py`, which is why `-e` works there — upstream's `setup.bash` uses `-e`
for both.

### Stage 5 complete — Neuralangelo

`requirements-blackwell.txt` changes:
- tiny-cuda-nn `master` → **`@cc-120`** (master's max architecture predates Blackwell)
- dropped `pathlib` — the PyPI package is an obsolete backport that shadows the stdlib module

uv reused the tcnn wheel built for the nerfstudio env, so this stage took seconds rather
than another full compile. Verified: `tcnn HashGrid OK -> (2048, 32)` on the 5090.

**Build-backend gotcha (2nd instance):** `gpustat==1.1.1` needs `setuptools_scm` but does
not declare it — same class as `fpsample`/`scikit_build_core` in Stage 3. Under
`--no-build-isolation` nothing supplies these. Pre-install the common backends into each
venv up front: `setuptools wheel ninja setuptools_scm scikit_build_core cmake pybind11
Cython numpy poetry-core hatchling flit_core`.

### Stage 6 — BundleSDF

Replaced the ~50 `sudo apt-get` packages **and** the four source builds (Eigen, OpenCV +
contrib, PCL, pybind11, yaml-cpp) with one conda-forge env, `r2s-bundlesdf`:
`eigen opencv pcl yaml-cpp pybind11 boost-cpp flann glog gflags hdf5 proj protobuf zeromq
cmake ninja`. No root, and it skips hours of compiling plus the `make -j$(nproc)` OOM risk.

**Two more hardcoded architectures found** — both would have built cleanly and then failed
at runtime on the 5090:
- `BundleTrack/CMakeLists.txt`:
  `set(CUDA_NVCC_FLAGS "... -gencode=arch=compute_86,code=sm_86 ...")` and
  `set(CMAKE_CUDA_ARCHITECTURES 52 60 61 70 75 80 86)` → both now target **120**.
- `mycuda/setup.py`: `nvcc_flags = [..., '-arch=sm_86']` → **`-arch=sm_120`**.

`BundleTrack/CMakeLists.txt` also did
`find_package(PCL REQUIRED PATHS ${CMAKE_PREFIX_PATH} NO_DEFAULT_PATH)` with
`CMAKE_PREFIX_PATH` pointing only at `../local` (where setup.bash's source builds landed).
Added an opt-in hook so the conda env can be used instead:
```cmake
if(DEFINED ENV{BUNDLESDF_DEPS_PREFIX})
  set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH};$ENV{BUNDLESDF_DEPS_PREFIX}")
endif()
```

Python pins relaxed: `numpy==1.26.1`, `Cython==0.29.20`, `scikit-image==0.17.2`,
`networkx==2.2` are all from ~2020 and will not build on cp310 with modern setuptools.
They are also **vestigial** — upstream's own `setup.bash` reinstalls unpinned
`scikit-image` and does `pip install --upgrade networkx` at the end, overwriting them.

**`uv venv` ships no pip.** Frosting got away with `python -m pip` only because its
requirements freeze happened to include `pip`; BundleSDF's venv did not, so
`.venv/bin/python -m pip` failed with `No module named pip`. Install pip explicitly into
any uv-created venv whose build steps shell out to it.

### Stage 6 — where the conda-forge substitution stops working

The conda-forge strategy replaced Eigen, PCL, yaml-cpp, pybind11, boost, flann, glog,
gflags, hdf5, proj, protobuf, zeromq/cppzmq, OpenGL/GLUT/GLEW and MPI successfully. It
fails for exactly one dependency:

**conda-forge's OpenCV has no CUDA modules** — 0 `cuda*` headers, 0 `libopencv_cuda*`
libraries. BundleTrack needs `cudafeatures2d.hpp`, `cudaimgproc.hpp` and
`cudaoptflow.hpp`. This is precisely why upstream's `setup.bash` compiles OpenCV +
opencv_contrib from source, and there is no way around it.

**OpenCV 4.12.0 specifically**: opencv_contrib **dropped `rgbd` and `xfeatures2d` after
4.12** (verified against the 4.13.x tree), and BundleTrack includes both
(`opencv2/rgbd.hpp`, `opencv2/xfeatures2d/nonfree.hpp`). 4.12 is also new enough to accept
`CUDA_ARCH_BIN=12.0`. `OPENCV_ENABLE_NONFREE=ON` is required for the SURF path.

Build is trimmed to the 17 modules BundleTrack actually references (traced from its
includes and `cv::cuda::` symbols) rather than all of OpenCV:
`core,imgproc,imgcodecs,highgui,videoio,calib3d,features2d,flann,video,cudev,cudaarithm,
cudawarping,cudaimgproc,cudafeatures2d,cudaoptflow,xfeatures2d,rgbd`, with tests, docs,
examples, python bindings and Java off.

**Link failure to expect:** `/usr/bin/ld: cannot find -lva / -lva-drm`. OpenCV picks up
VA-API *headers* from the conda env (it is on PATH for cmake) but the libraries are not
linkable. Fix: `-DWITH_VA=OFF -DWITH_VA_INTEL=OFF -DWITH_LIBVA=OFF`.

### Stage 6 — PCL 1.11+ smart-pointer migration

`pcl::PointCloud<T>::Ptr` changed from `boost::shared_ptr` to `std::shared_ptr` in PCL
1.11. BundleTrack is written against the older API, giving errors like:

```
Frame.cpp:103: no match for 'operator=' (operand types are
  'pcl::PointCloud<pcl::PointXYZRGBNormal>::Ptr' {aka 'std::shared_ptr<...>'} and
  'boost::shared_ptr<...>')
```
11 distinct sites, 53 error lines in `Bundler.cpp` and 26 in `Frame.cpp`.

Pinning PCL backwards does **not** work: PCL 1.8 uses `boost::shared_ptr` and needs no
patches, but it includes `boost/detail/endian.hpp`, removed in **Boost 1.69**. Old PCL and
modern Boost are mutually exclusive, so the source has to move forward.

Fix: replaced **52** occurrences of `boost::shared_ptr`/`boost::make_shared` with the
`std::` equivalents across `Utils.h`, `Utils.cpp`, `Frame.cpp`, adding `<memory>` where
missing. Every occurrence was a PCL type — nothing else in BundleTrack used boost smart
pointers — so the blanket replacement is safe. Bundler.cpp needed no edits; its errors
were downstream call sites.

### Stage 6 — other fixes

- **`cicc: not found`.** The deprecated `FindCUDA` module invokes nvcc such that it
  resolves its internal compiler `cicc` via `PATH` instead of relative to itself. Add
  `$CUDA_HOME/nvvm/bin` to `PATH` (note: on this conda layout `cicc` is at
  `$CUDA_HOME/nvvm/bin/cicc`, not under `targets/`).
- **PCL visualization / VTK.** A bare `find_package(PCL REQUIRED)` pulls in the
  `visualization` component, which hard-requires VTK. The only `pcl/visualization`
  includes in BundleTrack are commented out, so the fix is to request just the components
  used: `common io features filters octree recognition registration sample_consensus
  search kdtree`.
- **`AT_DISPATCH_FLOATING_TYPES(tensor.type(), ...)`** in `mycuda/common.cu` — `.type()`
  returns the deprecated `DeprecatedTypeProperties`; modern torch requires
  `.scalar_type()`. Three call sites.

### Stage 6 complete

```
torch      : 2.7.1+cu128  NVIDIA GeForce RTX 5090
my_cpp     : OK  (BundleTrack C++; exposes Bundler, Frame, GluNet, ...)
mycuda     : OK  (common, gridencoder)
kaolin     : 0.18.0   (all 13 APIs BundleSDF uses still exist)
pytorch3d  : 0.7.9
```

BundleTrack built to 100% with 0 errors. Note the Python-side `cv2` remains pip's
opencv-python (no CUDA) and that is correct: BundleSDF's Python code has **zero**
`cv2.cuda` uses, so only the C++ side needs the CUDA build, which it links directly.

Last fix of the stage: `FeatureManager.cpp` calls `pcl::geometry::distance` without
including `<pcl/common/geometry.h>`; PCL >= 1.11 no longer pulls it in transitively.

**Running it** — `my_cpp` and the shared libraries have to be reachable:
```bash
export LD_LIBRARY_PATH=<BundleSDF>/local/lib:<deps env>/lib:<cuda env>/lib:$LD_LIBRARY_PATH
export PYTHONPATH=<BundleSDF>:<BundleSDF>/BundleTrack/build:$PYTHONPATH
```

`setup_blackwell.bash` now reproduces all of stage 6 from scratch and is verified end
to end.

---

## Port summary

All six stages complete. What the port actually consisted of:

| Category | Count | Instances |
|---|---|---|
| Hardcoded CUDA architectures | 4 | tiny-cuda-nn (`cc-120` branch), `BundleTrack/CMakeLists.txt` (`52 60 61 70 75 80 86`), `mycuda/setup.py` (`-arch=sm_86`), OpenCV (`CUDA_ARCH_BIN`) |
| Dead or unreachable pins | 4 | drake nightly pruned + no cp310; pytorch3d `cu121` wheel; pytorch3d `cu118` wheel; opencv_contrib `rgbd`/`xfeatures2d` gone after 4.12 |
| Toolchain-era breakage | 5 | GCC 13 dropped transitive `<cstdint>`; conda CUDA header/lib64 layout; PCL 1.11 `boost::`→`std::shared_ptr` (52 sites); `pcl/common/geometry.h`; torch `AT_DISPATCH` `.type()`→`.scalar_type()` |
| Packaging / tooling traps | 7 | Poetry keyring/DBus; Poetry dulwich; editable `simple-knn`; 2x undeclared build backends; `uv venv` has no pip; VA-API link failure |
| Genuinely unavailable | 1 | conda-forge OpenCV has no CUDA modules — the one unavoidable source build |

Only the first row is about Blackwell. Everything else is eighteen months of drift in a
research repo plus a distro newer than the one it was written for.

**Environments produced**

| Path | Contents |
|---|---|
| `~/micromamba/envs/r2s-cuda` | nvcc 12.8 (build-time toolkit) |
| `~/micromamba/envs/r2s-colmap` | COLMAP 4.2.0 with CUDA |
| `~/micromamba/envs/r2s-bundlesdf` | C++ deps for BundleTrack |
| `.venv` | core: SAM2, asset generation, robot_payload_id |
| `.venv_nerfstudio` | nerfstudio 1.1.5, gsplat 1.4.0, tinycudann 1.7 |
| `Frosting/.venv` | pytorch3d 0.7.9, rasterizers, nvdiffrast |
| `neuralangelo/.venv` | tinycudann |
| `BundleSDF/.venv` + `BundleSDF/local` | kaolin, mycuda, my_cpp; CUDA OpenCV 4.12.0 |

**Forks** (each carries a `blackwell-port` branch): `Chung-I/scalable-real2sim`,
`Chung-I/robot_payload_id`, `Chung-I/Frosting`, `Chung-I/neuralangelo`,
`Chung-I/BundleSDF`.

**Not done** (out of scope so far): LoFTR `outdoor_ds.ckpt` weights (manual Google Drive
download), the 71 GB benchmark dataset beyond the 70 MB `robot_system_id_data` already
fetched, and an end-to-end `run_asset_generation.py` run.

---

## End-to-end smoke run — what only surfaced at runtime

Component tests (every extension imports, every kernel runs) passed while **five** separate
problems remained. All of these needed an actual pipeline run to find.

**1. neuralangelo's nested `third_party/colmap` submodule was never initialised.**
`convert_data_to_json.py` does
`from third_party.colmap.scripts.python.read_write_model import ...`, which fails at
*import* of `run_asset_generation.py`. The neuralangelo step in the README has a second
`git submodule init && git submodule update` that is easy to miss. `git submodule update
--depth 1` fails here with `fatal: transport 'file' not allowed`; fetch the pinned commit
directly instead:
```bash
git init . && git remote add origin https://github.com/colmap/colmap.git
git fetch --depth 1 origin 43de802cfb3ed2bd155150e7e5e3e8c8dd5aaa3e && git checkout FETCH_HEAD
```

**2. torchvision's bundled libjpeg shadows conda's.**
```
ImportError: .../r2s-bundlesdf/lib/libtiff.so.6: undefined symbol: jpeg12_write_raw_data
```
conda's `libjpeg.so.8` *does* export the 26 `jpeg12_*` symbols and resolves correctly on
its own. But `torchvision.libs/libjpeg.*.so.8` has **SONAME `libjpeg.so.8`** too and lacks
them, and `bundlesdf.py` imports torch before `my_cpp`, so the loader reuses torchvision's
copy. Fix: `export LD_PRELOAD=$DEPS_ENV/lib/libjpeg.so.8`.

**3. Trimming upstream's pip list removed load-bearing packages.**
`dearpygui` (gui.py), `pymeshlab`, `yacs` (LoFTR config). Resolve the import chain locally
(`python -c "import bundlesdf"`) rather than one full pipeline launch per missing module.

**4. `--use_gui 1` hangs headless.** `run_asset_generation.py` passes it to
`run_custom.py`, and `bundlesdf.py` then spawns a dearpygui window as a
`multiprocessing.Process`. With no display it never signals started and the run blocks
forever. Set `--use_gui 0`. Note installing `dearpygui` turns a clean crash into a silent
hang, so fix 3 makes this one *harder* to see.

**5. `mycuda` must be installed EDITABLE — the inverse of simple-knn.**
`mycuda/setup.py` declares top-level extensions `common` and `gridencoder`, but `Utils.py`
does `from mycuda import common`. A non-editable install puts `common.so` in site-packages
as a top-level module and leaves `mycuda/` without it:
```
ImportError: cannot import name 'common' from 'mycuda' (unknown location)
```
That import sits in a `try/except` which swallows it, a NeRF worker process then dies, and
the parent blocks on its pipe **forever at 0% CPU and 0% GPU with no error printed**.

Diagnosing it needed kernel-level evidence, not the log: main thread in
`poll_schedule_timeout`, all worker threads in `futex_do_wait`, and a **zombie child**.
(`py-spy dump` is blocked by ptrace_scope without root; `/proc/<pid>/task/*/wchan` and
`ps --ppid` work fine.)

Worth stating plainly: **simple-knn must be non-editable** (no `__init__.py`, so the
editable finder maps nothing) while **mycuda must be editable** (the code imports it as a
package). Upstream's setup.bash uses `-e` for both; correcting that uniformly breaks one
or the other.

---

## Validation: inertial parameters vs upstream's own reference

`inflator.tar` ships upstream's `bundle_sdf_inertial_params.json`, so the smoke run is a
*validation*, not just a smoke test. Ours came from a **360-frame** reconstruction (every
5th frame; the full 1800-frame merge OOMs on a 30 GB box), theirs from the full set.

| quantity | ours | reference | difference |
|---|---|---|---|
| mass (kg) | 0.654720 | 0.654722 | **0.00 %** |
| \|CoM\| (mm) | 34.71 | 34.29 | 1.24 % |
| inertia Frobenius | 0.045488 | 0.045465 | 0.05 % |
| inertia eigenvalues | 3.0e-6, 0.032164, 0.032165 | 3.0e-6, 0.032148, 0.032148 | **0.05 %** |

The raw CoM *looks* wrong — y and z have opposite signs, 68 mm apart. That is a
**canonical-frame convention**, not an error: rotating the reference 180 degrees about x
drops the residual from **68.46 mm to 2.08 mm**. The mesh canonicalisation step landed in
a differently-oriented frame for our shorter reconstruction, which flips y and z.

The frame-independent quantities are the ones that matter, and they agree:
- mass is frame-independent -> matches to 6 significant figures
- inertia **eigenvalues** are rotation-invariant -> 0.05 %
- CoM magnitude is rotation-invariant -> 1.24 %

So the identification pipeline (arm-alone parameters, arm+object parameters, subtraction
under the pseudo-inertia constraint) reproduces upstream's numbers on Blackwell, using a
5x-decimated reconstruction. Mass and inertia are insensitive to the mesh detail; only the
frame convention moved.

### Final blocker before this ran

`robot_payload_id/utils/utils.py` locates its Drake models via
`Path(__file__).parent.parent.parent / "models" / "package.xml"`. That arithmetic assumes a
**source tree**. Poetry installs the path dependency as a *copy* into site-packages, where
three levels up is `site-packages/` itself, so it fails with
`XML_ERROR_FILE_NOT_FOUND`. Fix: install it editable
(`uv pip install --no-deps -e scalable_real2sim/robot_payload_id`).

That is the **third** editable-vs-copy decision in this port, and all three need different
answers:

| package | required mode | why |
|---|---|---|
| `simple-knn` | **non-editable** | declares no `packages=` and has no `__init__.py`, so the editable finder maps nothing |
| `mycuda` | **editable** | code does `from mycuda import common`, needing the .so built in place |
| `robot_payload_id` | **editable** | resolves a data directory relative to `__file__` |

Also note the first run of the identification pulls **drake_models** (a large tarball from
github.com/RobotLocomotion/models) into `~/.cache/drake`; it downloaded at ~0.5 MB/s here
and retried a mirror once, so expect several minutes before any output appears.
