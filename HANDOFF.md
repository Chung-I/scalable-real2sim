# Handoff: Scalable Real2Sim on the RTX 5090

Written 2026-09-22 for the next agent. Read this file first. Read `PORTING_NOTES.md`
for the reasons behind each fix. Do not repeat work that this file marks as done.

## 1. State in one paragraph

The upstream repo (nepfaff/scalable-real2sim) targets Ubuntu 22.04, CUDA 12.1 and
torch 2.3.1. None of that runs on an RTX 5090 (compute capability 12.0, "Blackwell").
This machine now has a complete port. Every environment builds, every CUDA kernel runs
on sm_120, and the pipeline reproduces the authors' own inertial parameters for one
object to 0.00 % on mass. The full end-to-end run through `run_asset_generation.py` is
**not** yet complete. Section 6 lists what is open.

## 2. Machine facts that decide how you work

| Fact | Consequence |
|---|---|
| RTX 5090, driver 580.173.02 | Only torch >= 2.7 with cu128 has sm_120 kernels. |
| 30 GB RAM, 8 GB swap | Heavy jobs must run in a memory-capped systemd scope (section 4). |
| No passwordless `sudo` | Everything system-level comes from micromamba, never apt. |
| Ubuntu 24.04, GCC 13.3 | Old C++ that relied on transitive `<cstdint>` includes breaks. |

**CAUTION: Never use `pkill -f` or `pgrep -f` with a pattern that appears in your own
command.** It matches itself. Use `ps -eo cmd | grep -c "[r]un_custom"` instead.

**CAUTION: Never run a heavy job outside a systemd scope.** An OOM kill then takes the
whole tmux scope, including the agent session. This happened before on this machine.

## 3. Where everything is

Parent repo: `~/Codes/scalable-real2sim`, branch `blackwell-port`, commit `fb52349`.
Remote `fork` is `Chung-I/scalable-real2sim`. Remote `origin` is upstream. Push to
`fork` only.

Submodules, each on its own `blackwell-port` branch in a `Chung-I/*` fork:

| Submodule | Commit | What changed |
|---|---|---|
| `scalable_real2sim/BundleSDF` | `29b24fd` | sm_120 targets, PCL 1.14 migration, trimesh 5 migration, `setup_blackwell.bash` |
| `scalable_real2sim/Frosting` | `6a96161` | `<cstdint>` includes, `requirements-blackwell.txt` |
| `scalable_real2sim/neuralangelo` | `684ace9` | `requirements-blackwell.txt` (tcnn `cc-120` branch) |
| `scalable_real2sim/robot_payload_id` | `f027c0a` | drake pin relaxed to `>=1.41.0,<1.52.0` |

`.gitmodules` already points at the forks. A fresh clone with `--recurse-submodules`
gets the ported code.

Environments (all built, all verified):

| Path | Size | Purpose |
|---|---|---|
| `~/micromamba/envs/r2s-cuda` | 5.3 GB | nvcc 12.8. Build-time toolkit. |
| `~/micromamba/envs/r2s-colmap` | 3.9 GB | COLMAP 4.2.0 with CUDA. |
| `~/micromamba/envs/r2s-bundlesdf` | 2.8 GB | C++ deps for BundleTrack (PCL 1.14, Eigen, yaml-cpp, cppzmq, GL). |
| `.venv` | 9.2 GB | Core Poetry env. SAM2, drake 1.41.0, `robot_payload_id`. Runs `run_asset_generation.py`. |
| `.venv_nerfstudio` | 9.2 GB | nerfstudio 1.1.5, gsplat 1.4.0, tinycudann 1.7. |
| `scalable_real2sim/Frosting/.venv` | 994 MB | pytorch3d 0.7.9, rasterizers, nvdiffrast. |
| `scalable_real2sim/neuralangelo/.venv` | 368 MB | tinycudann. |
| `scalable_real2sim/BundleSDF/.venv` | 1.3 GB | kaolin 0.18.0, mycuda, my_cpp. |
| `scalable_real2sim/BundleSDF/local` | 211 MB | Our OpenCV 4.12.0 with CUDA. The one unavoidable source build. |

Data and results:

| Path | Content |
|---|---|
| `data/hf/scalable_real2sim_benchmark_dataset/robot_system_id_data/` | Arm identification, 70 MB. All 5 gripper openings. |
| `data/hf/scalable_real2sim_benchmark_dataset/object_data/inflator.tar` | One object, 3.6 GB. Downloaded. |
| `data/smoke/object_data/inflator/` | Extracted, 1800 frames. Too large for 30 GB RAM. |
| `data/smoke/object_data_small/inflator/` | Every 5th frame, 360 frames. This is what ran. |
| `data/smoke/out_small/inflator/bundle_sdf_mesh/textured_mesh.obj` | Our mesh, 3.4 MB. |
| `data/smoke/OURS_bundle_sdf_inertial_params.json` | Our result. |
| `data/smoke/REFERENCE_bundle_sdf_inertial_params.json` | The authors' result, from the tar. |
| `scalable_real2sim/BundleSDF/BundleTrack/LoFTR/weights/outdoor_ds.ckpt` | Downloaded. Required at run time. |

`data/` is git-ignored. It exists only on this machine.

## 4. How to run

Procedure. Each command is one step.

1. Change to the repo root: `cd ~/Codes/scalable-real2sim`.
2. Export the environment. Every line is required. See section 7 for why.

```bash
export CUDA_HOME=$HOME/micromamba/envs/r2s-cuda
export DEPS=$HOME/micromamba/envs/r2s-bundlesdf
export PATH=$HOME/micromamba/envs/r2s-colmap/bin:$CUDA_HOME/bin:$CUDA_HOME/nvvm/bin:$PATH
export TORCH_CUDA_ARCH_LIST="12.0"
export LD_PRELOAD=$DEPS/lib/libjpeg.so.8
```

3. Launch detached, inside a memory-capped scope, with absolute log paths:

```bash
setsid nohup bash -c '
cd /home/chungyili/Codes/scalable-real2sim
systemd-run --user --scope -p MemoryMax=24G -p MemorySwapMax=8G -- \
  ./.venv/bin/python run_asset_generation.py \
    --data-dir data/smoke/object_data_small \
    --robot-id-dir data/hf/scalable_real2sim_benchmark_dataset/robot_system_id_data \
    --output-dir data/smoke/out_small \
    --skip-segmentation \
  > /home/chungyili/Codes/scalable-real2sim/run.log 2>&1
printf "\nDONE_EXIT=%s\n" "$?" >> /home/chungyili/Codes/scalable-real2sim/run.log' &
```

4. Make sure that the job started. Count tracked frames, not log lines:
   `ls data/smoke/out_small/inflator/bundle_sdf/ob_in_cam | wc -l`.
5. If the count stops and the process is alive at 0 % CPU, the job is hung. Read section 7.
6. When the job ends, look for `DONE_EXIT=` in `run.log`. If the line is absent and the
   process is gone, the OOM killer took it. Confirm with
   `journalctl --user -n 20 | grep -i oom`.

Notes:
- `--data-dir` is a parent directory. The script processes every subdirectory in it.
- `--skip-segmentation` works because the tar ships `masks/` and `gripper_masks/`.
- BundleSDF alone takes about 21 minutes for 360 frames.
- Do not use the 1800-frame directory. It ran out of memory in the point-cloud merge.

## 5. What is verified

| Check | Result |
|---|---|
| torch 2.7.1+cu128 on the GPU | capability (12, 0), `sm_120` in arch list, matmul runs |
| tinycudann HashGrid + FullyFusedMLP | forward and backward, fp16 |
| pytorch3d `knn_points` | CUDA kernel runs |
| Frosting rasterizers, simple-knn `distCUDA2` | CUDA kernels run |
| kaolin | all 13 APIs that BundleSDF calls exist and run |
| BundleTrack `my_cpp` | imports with a clean environment, exposes `Bundler`, `Frame`, `GluNet` |
| BundleSDF on 360 frames | tracked 1800/1800 and 360/360 frames, produced a textured mesh |
| Inertial identification on that mesh | see the table below |

Our result against the authors' reference for the inflator:

| Quantity | Ours (360 frames) | Reference (1800 frames) | Difference |
|---|---|---|---|
| mass (kg) | 0.654720 | 0.654722 | 0.00 % |
| inertia eigenvalues | 3.0e-6, 0.032164, 0.032165 | 3.0e-6, 0.032148, 0.032148 | 0.05 % |
| CoM magnitude (mm) | 34.71 | 34.29 | 1.24 % |

The raw CoM vectors differ by 68 mm because our mesh frame is rotated 180 degrees about
x. After that rotation the residual is 2.08 mm. Compare rotation-invariant quantities
only.

## 6. What is open

1. **No complete run through `run_asset_generation.py`.** The last run stopped at stage 5
   with `XML_ERROR_FILE_NOT_FOUND`. The cause is fixed (`robot_payload_id` is now
   installed editable). Stage 5 was then run directly on the stage-3 mesh and succeeded.
   A fresh full run is unblocked and takes about 25 minutes to reach stage 5.
2. **Nerfacto and Frosting stages never ran on real data.** They are the stages after 5.
   Expect new problems there. Each runs in its own venv.
3. **1800-frame reconstruction needs more than 30 GB RAM.** The point-cloud merge after
   tracking holds every frame's cloud at once. A streaming merge would fix it. Not done.
4. **MOSEK is not licensed.** Drake falls back to SCS, CSDP or Clarabel for the SDP. It
   worked on the inflator. For exact paper numbers, a free academic license goes at
   `~/mosek/mosek.lic`. No code change is needed.
5. **An unresolved question from the user.** A teammate session asked for simulator
   throughput. I quoted the global CLAUDE.md figure of 6.6K aggregate FPS for ManiSkill3
   on the NCHC H200. The user replied "no we don't have it" and did not say which claim
   was wrong. If that figure is wrong, the teammate needs a correction.

## 7. Things that bite immediately

These are the failures a new agent hits in the first hour. Each has a one-line fix.

| Symptom | Cause | Fix |
|---|---|---|
| `libtiff.so.6: undefined symbol: jpeg12_write_raw_data` on `import my_cpp` | torchvision bundles its own `libjpeg.so.8` and loads first | `export LD_PRELOAD=$DEPS/lib/libjpeg.so.8` |
| Run hangs at 0 % CPU after "translation [...]" | `--use_gui 1`, or `mycuda` installed non-editable | Already fixed in `fb52349` and `29b24fd`. Do not reinstall `mycuda` without `-e`. |
| `ImportError: cannot import name 'common' from 'mycuda'` | non-editable install | `pip install --no-build-isolation -e scalable_real2sim/BundleSDF/mycuda` |
| `import simple_knn` fails although the `.so` exists | editable install | Install simple-knn **non-editable**. It is the opposite of mycuda. |
| `PackageMap ... models/package.xml ... FILE_NOT_FOUND` | `robot_payload_id` installed as a copy | `uv pip install --python .venv/bin/python --no-deps -e scalable_real2sim/robot_payload_id` |
| `fatal error: cuda_runtime_api.h` in any CUDA build | conda CUDA layout | `ln -sfn $CUDA_HOME/targets/x86_64-linux/include/* $CUDA_HOME/include/` |
| `sh: 1: cicc: not found` in a cmake build | FindCUDA looks up cicc on PATH | put `$CUDA_HOME/nvvm/bin` on PATH |
| `No module named 'third_party.colmap.scripts'` at import | neuralangelo nested submodule | fetch colmap at `43de802c` into `neuralangelo/third_party/colmap` |
| Poetry `AssertionError` in `partial_solution.py` | unsatisfiable constraint, reported badly | run the same constraints through `uv pip compile` to see the real conflict |
| Poetry stops with a DBus keyring error | no secret-service session | `poetry config keyring.enabled false` |

## 8. How to rebuild from nothing

BundleSDF: `bash scalable_real2sim/BundleSDF/setup_blackwell.bash`. It is verified end to
end and idempotent. It takes 1 to 2 hours, mostly the OpenCV build.

Everything else: follow the stage order in `PORTING_NOTES.md`. The core env is Poetry
(`poetry lock` then `poetry install`, keyring off, `system-git-client true`). The other
venvs are `uv venv --python 3.10` plus the commands recorded in the notes.

## 9. Related records

- `PORTING_NOTES.md` in this repo: every fix, its error text, its cause, its reason.
- `~/Codes/daily-logs/2026-09-06.md`: paper summaries and the first arm-identification check.
- `~/Codes/daily-logs/2026-09-22.md`: plain-words summary of the whole effort.
- Global `~/.claude/CLAUDE.md`: machine rules. The OOM scope rule and the `pgrep` rule
  are there. Obey them.
