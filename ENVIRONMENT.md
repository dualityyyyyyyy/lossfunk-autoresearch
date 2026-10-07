# Reproducible PushT Execution Environment

**Status:** local software validation is complete, but the local machine cannot run the official CUDA-configured baseline. Remote Linux execution with an NVIDIA GPU is required. No training, baseline reproduction, or scientific experiment has been run.

## Platform decision

The native Windows host is Windows 11 Pro, build 26200. The repository setup is Linux-oriented (Bash activation/data scripts, `MUJOCO_GL=egl`, and native simulation/rendering dependencies), and the baseline evaluator hard-codes CUDA. Direct Windows is therefore not the reproducible target.

I selected the already-installed **WSL2 Ubuntu** distro for isolated environment preparation. The Windows host has an AMD integrated Radeon 610M and no NVIDIA GPU or NVIDIA driver; WSL has `/dev/dxg` but no `nvidia-smi`, and the installed PyTorch builds are CPU-only. The official LeWM evaluator hard-codes `model.to("cuda")` and its checked-in CEM config uses `device: cuda`; the unchanged evaluator therefore cannot run locally. The 0.1.0 candidate passed loader/checkpoint/world/data checks but could not initialize the configured CUDA model or planner. Remote Linux execution with an NVIDIA GPU is required. No CPU substitution was attempted. No provider has been selected.

The venv was first placed at the workspace root as requested. Imports then stalled in uninterruptible I/O on the Windows-mounted `/mnt/c` filesystem. The canonical venv is consequently at the WSL-native path below; setup scripts and lock/artifact files remain in the shared workspace. This is the documented reason the preferred in-workspace venv location was not retained as canonical.

## Host and runtime inventory

| Item | Observed value |
|---|---|
| Windows host | Microsoft Windows 11 Pro, 10.0.26200, 64-bit |
| WSL | WSL 2; kernel `6.6.87.2-microsoft-standard-WSL2` |
| WSL distribution | Ubuntu 26.04 LTS (`resolute`) |
| CPU visible to WSL | 8 logical CPUs; model name was not available to the restricted WMI query |
| Memory visible to WSL | 7,008,480 kB (about 6.7 GiB) |
| GPU | AMD Radeon(TM) 610M, integrated; Windows reported 2 GiB adapter memory |
| NVIDIA/CUDA | No NVIDIA GPU; `nvidia-smi` and `nvcc` absent in WSL; `torch.cuda.is_available()` is `False` |
| WSL root filesystem | 1,007 GiB total, 941 GiB available at inventory |
| Windows `C:` mounted in WSL | 396 GiB total, 125 GiB available at inventory |
| Windows Python | Python 3.14.3 selected; Python Launcher also listed 3.12 and 3.11 |
| WSL system Python | Python 3.14.4; not used for the project environment |
| Project Python | Managed CPython 3.10.21, installed by uv; isolated venv uses this interpreter |
| Environment manager | No uv or conda was initially available in WSL. A pinned uv binary is kept under `.research_env/bin/uv`; no global Python packages were changed. |

WSL commands sometimes emitted `Failed to start the systemd user session for 'bhavana'` while still completing. After a server restart, the restricted Codex identity also received `E_ACCESSDENIED` from WSL; the already-authorized host identity could query and use the distro. This was an execution-context issue, not a package or distro failure.

## Repository revisions

| Repository | Commit |
|---|---|
| `https://github.com/lucas-maes/le-wm.git` checkout | `8edfeb336732b5f3ce7b8b210d0ba370a09e2cac` |
| `https://github.com/NiccoloCase/Hi-LeWM.git` checkout | `4bb21a2888e8f22b8d084762c80361e398968775` |
| Hi-LeWM `third_party/lewm` baseline submodule | `83f97d72ad067855bc89a1b74b4aff11d4dfdf0c` |

No source files in either upstream checkout were modified. The Hi-LeWM submodule was initialized as required by its documented wrapper. Hydra overrides described below are invocation-level integration settings, not source edits.

## Environment paths and tools

- Shared workspace, as seen from WSL: `/mnt/c/Users/DELL/lossfunk_autoresearch`
- Workspace-local setup tools, cache, constraints, and checkpoint artifacts: `$WORKSPACE/.research_env/`
- Canonical venv: `/home/bhavana/lossfunk_autoresearch/.venv-linux`
- Managed CPython: `/home/bhavana/lossfunk_autoresearch/.research_env/python/`
- uv: `$WORKSPACE/.research_env/bin/uv`, version `0.12.20`
- uv archive SHA256: `6590717592ace991ff83a63fef799e3ad9d33ecc8f96c5d6bdd732496e79337f`
- Exact installed package freeze: [ENVIRONMENT.lock.txt](ENVIRONMENT.lock.txt) (177 distributions)
- Resolver constraints: `.research_env/constraints-cpu.txt`

The lock records the installed versions. The important compatibility pins are:

| Package | Installed version | Reason |
|---|---:|---|
| Python | 3.10.21 | Repository-documented Python 3.10; exact managed patch recorded. |
| `torch` | 2.14.1+cpu | CPU wheel; this host has no CUDA-capable NVIDIA GPU. |
| `torchvision` | 0.29.1+cpu | Matching CPU wheel. |
| `stable-worldmodel` | 0.0.6 | Pinned for strict instantiation/conversion of the official LeWM weights in the prepared environment. It is **not API-compatible with the standalone LeWM evaluator** at the checked-out commit: this PyPI artifact does not include `stable_worldmodel.wm.utils`, which `le-wm/eval.py` calls. |
| `stable-pretraining` | 0.1.6 | Contemporaneous ViT implementation; strict checkpoint state-dict loading passed. |
| `transformers` | 4.57.6 | Resolved compatible version in the frozen dependency set. |
| `datasets` | 2.14.4 | Avoids the incompatible 1.1.1 version initially selected. |
| `pyarrow` | 20.0.0 | Compatible with the selected dataset stack and `stable-pretraining` constraints. |
| `gymnasium` | 1.3.0 | PushT environment registration/runtime. |
| `mujoco` | 3.14.0 | Environment dependency installed by the repository extras. |
| `hydra-core` | 1.3.7 | Config composition and evaluator entrypoints. |
| `swig` | 4.5.0 | Isolated build tool needed by transitive `box2d-py` source build. |

The installation uses the repositories’ documented `stable-worldmodel[train,env]` extra, which brings a broad transitive environment/training dependency set. There is no PushT-only extra in the inspected setup. The full set is frozen in `ENVIRONMENT.lock.txt`; no CUDA/NVIDIA packages appear in that final lock.

## Recreate the environment

Run these commands from WSL Ubuntu. Substitute a task-specific native WSL directory for `WSL_PROJECT` on another machine. Keep the venv and uv cache on WSL’s Linux filesystem; keep source and recorded artifacts in the shared workspace.

```bash
WORKSPACE=/mnt/c/Users/DELL/lossfunk_autoresearch
WSL_PROJECT=/home/bhavana/lossfunk_autoresearch
UV_BIN="$WORKSPACE/.research_env/bin/uv"
VENV="$WSL_PROJECT/.venv-linux"
CPU_INDEX=https://download.pytorch.org/whl/cpu
PYPI_INDEX=https://pypi.org/simple

mkdir -p "$WSL_PROJECT/.research_env/python" "$WSL_PROJECT/.cache"
export UV_PYTHON_INSTALL_DIR="$WSL_PROJECT/.research_env/python"
export UV_CACHE_DIR="$WORKSPACE/.research_env/cache"
export UV_LINK_MODE=copy

# If uv is not already in the workspace, fetch the pinned Linux release:
mkdir -p "$WORKSPACE/.research_env/bin"
curl -fL -o "$WORKSPACE/.research_env/uv.tar.gz" \
  https://github.com/astral-sh/uv/releases/download/0.12.20/uv-x86_64-unknown-linux-gnu.tar.gz
echo '6590717592ace991ff83a63fef799e3ad9d33ecc8f96c5d6bdd732496e79337f  '"$WORKSPACE/.research_env/uv.tar.gz" | sha256sum -c -
tar -xzf "$WORKSPACE/.research_env/uv.tar.gz" --strip-components=1 -C "$WORKSPACE/.research_env/bin"
chmod +x "$UV_BIN"

"$UV_BIN" venv --python 3.10.21 "$VENV"

# Make SWIG available to isolated builds of the transitive box2d-py package.
"$UV_BIN" pip install --python "$VENV/bin/python" \
  --index-url "$CPU_INDEX" --extra-index-url "$PYPI_INDEX" \
  --index-strategy unsafe-best-match 'swig==4.5.0'
export PATH="$VENV/bin:$PATH"

# Exact package versions used for this setup are frozen in the workspace lock.
"$UV_BIN" pip sync --python "$VENV/bin/python" \
  --index-url "$CPU_INDEX" --extra-index-url "$PYPI_INDEX" \
  --index-strategy unsafe-best-match "$WORKSPACE/ENVIRONMENT.lock.txt"

"$VENV/bin/python" --version
"$UV_BIN" pip check --python "$VENV/bin/python"
```

The constraints used when initially resolving the documented extras are:

```text
torch==2.14.1+cpu
torchvision==0.29.1+cpu
datasets>=2.0.0
stable-worldmodel==0.0.6
stable-pretraining==0.1.6
```

The corresponding file is `.research_env/constraints-cpu.txt`. For a fresh resolution instead of syncing the frozen package file, use `uv pip install ... --constraint "$WORKSPACE/.research_env/constraints-cpu.txt" 'stable-worldmodel[train,env]'` with the same two indexes and `unsafe-best-match` strategy, then capture a new freeze. Do not use the latest unpinned library versions: those failed the checkpoint compatibility check below.

## PushT checkpoint artifact

The single downloaded model is the documented official LeWM PushT checkpoint from `https://huggingface.co/quentinll/lewm-pusht`:

- `weights.pt`: 72,290,721 bytes; SHA256 `48938400ae3464c9680731287f583a9cb516f55a8ec64ea13a91be47fb15b607` (matches the repository page’s published file hash).
- `config.json`: 1,313 bytes.
- Converted object checkpoint: `.research_env/checkpoints/pusht/lewm_object.ckpt`, SHA256 `91825ebcc183f6119deae85d0c98d82b68b1c0f151769c9d2ba8544b3e2796a0`.
- Cache root: set `STABLEWM_HOME="$WORKSPACE/.research_env/checkpoints"`.

The repository converter passed a strict state-dict load with 303 keys, zero missing keys, and zero unexpected keys under `stable-worldmodel==0.0.6` and `stable-pretraining==0.1.6`. This verifies model construction and weights against that class layout; it does not verify the standalone LeWM evaluator's checkpoint-loader API, package cache layout, or inference path:

```bash
cd "$WORKSPACE/Hi-LeWM"
export PYTHONPATH="$PWD/third_party/lewm:$PWD"
export STABLEWM_HOME="$WORKSPACE/.research_env/checkpoints"
"$VENV/bin/python" scripts/convert_hf_weights_to_object_ckpt.py \
  --weights "$STABLEWM_HOME/pusht/weights.pt" \
  --config "$STABLEWM_HOME/pusht/config.json" \
  --run-name pusht/lewm \
  --cache-dir "$STABLEWM_HOME"
```

No Hi-LeWM hierarchical checkpoint was downloaded. Hi-LeWM training and full evaluation remain unvalidated. The official PushT HDF5 dataset was later downloaded and minimally checked; see **Shared PushT dataset cache** below.

## Source compatibility adjustments needed at invocation time

The current `stable-worldmodel==0.0.6` PushT constructor rejects `history_size` and `frame_skip` when Hi-LeWM’s evaluator forwards those config entries as environment kwargs. The non-mutating Hydra deletion overrides below were validated for environment construction:

```text
~world.history_size ~world.frame_skip
```

For Hi-LeWM checkpoint deserialization, keep the pinned baseline source on `PYTHONPATH`:

```bash
export PYTHONPATH="$WORKSPACE/Hi-LeWM/third_party/lewm:$WORKSPACE/Hi-LeWM"
```

Without that path, `AutoCostModel` cannot unpickle the converter’s `jepa.JEPA` class (`ModuleNotFoundError: No module named 'jepa'`). This environment variable and the Hydra deletion overrides avoid editing upstream source.

The validated CPU override fields are `solver.device=cpu` for flat planning and `planning.high.solver.device=cpu planning.low.solver.device=cpu` for hierarchy. The README evaluation commands were not run; a complete run also requires `pusht_expert_train` in the configured data cache.

## Validation performed

Only software setup checks were performed:

| Check | Outcome |
|---|---|
| `uv pip check` on final venv | Passed: all 177 installed packages compatible. |
| Core imports (`torch`, `torchvision`, Gymnasium, MuJoCo, Hydra, stable-worldmodel, stable-pretraining, Hi-LeWM modules) | Passed under final pins. |
| Hydra PushT evaluation config composition with CPU/flat overrides | Passed. |
| CEM solver initialization and configuration | Passed on CPU; no `solve()` call. |
| PushT world creation and reset | Passed with the two unsupported config fields removed; one environment, no action. |
| LeWM PushT model initialization | Passed. Strict checkpoint loading into pinned LeWM JEPA passed. |
| Hi-LeWM low-level checkpoint loader and `AutoCostModel` | Passed with baseline path on `PYTHONPATH`. |
| Flat `WorldModelPolicy` construction and environment attachment | Passed on CPU; no plan solve. |
| Hi-LeWM model assembly and one synthetic zero-latent forward | Passed; output shape `(1, 1, 192)`. This is an interface shape check, not a scientific prediction. |
| Hierarchical policy construction; high/low CEM configuration; PushT reset | Passed on CPU; neither solver was run and no action was taken. |
| Official PushT dataset path/readability check | Passed: `STABLEWM_HOME` resolved to `/home/bhavana/.stable_worldmodel`; the HDF5 file opened with `h5py` and its root keys were listed. No trajectories or evaluator rows were read. |
| Base checkpoint in shared cache | Passed: the cached object checkpoint SHA256 matches the previously recorded workspace checkpoint hash. No model inference or evaluator run occurred. |
| LeWorldModel standalone evaluator loader | Failed before checkpoint resolution: checked-out `le-wm/eval.py` expects `stable_worldmodel.wm.utils.load_pretrained`, absent from installed `stable-worldmodel==0.0.6`. The package/API compatibility investigation is recorded in `RUN_LOG.md`; `stable-worldmodel==0.1.0` is a candidate API release, not yet installed or validated here. |

No checkpoint-to-environment policy action, trajectory/sample read, task return, task success, or completed full evaluation occurred. The later baseline attempt constructed the environment and loaded dataset columns, then stopped at the incompatible evaluator loader; see `RUN_LOG.md`. Opening the HDF5 container and listing root keys was a path/readability check only. These checks do not establish that a scientific run is performant or reproducible.

## Setup failures and resolutions

| Failure | Resolution / current status |
|---|---|
| Initial dependency resolution selected CUDA 13 PyTorch plus several GiB of NVIDIA libraries despite no NVIDIA GPU. Install was interrupted before completion. | Use the official PyTorch CPU index, pin `torch==2.14.1+cpu` and `torchvision==0.29.1+cpu`, and explicitly resolve across the trusted PyTorch/PyPI indexes. The final lock contains no NVIDIA packages. |
| A first CPU-index attempt still selected PyPI’s CUDA torch because uv’s default first-index strategy chose PyPI. | Use `--index-strategy unsafe-best-match` with explicit CPU/PyPI indexes and exact CPU torch constraints. |
| `box2d-py==2.3.5` build failed because isolated build could not find `swig`. | Install `swig==4.5.0` into the isolated venv and put `$VENV/bin` on `PATH`; the subsequent build passed. No WSL system package was installed. |
| Initial version resolution selected `datasets==1.1.1` with `pyarrow==24.0.0`; importing stable-pretraining failed because `PyExtensionType` was absent. | Constrain datasets to at least 2; the compatible contemporaneous package pins resolve to `datasets==2.14.4`, `pyarrow==20.0.0`. Imports pass. |
| Converter/checkpoint strict loading failed with `stable-worldmodel==0.1.1` and `stable-pretraining==0.1.7`; those releases construct a newer ViT/LeWM state layout than the published weights. | Pin `stable-worldmodel==0.0.6` and `stable-pretraining==0.1.6`; the repo converter then loaded strictly (303/303 keys). |
| Hi-LeWM `AutoCostModel` could not unpickle `jepa.JEPA` with only the project root on `PYTHONPATH`. | Add `Hi-LeWM/third_party/lewm` to `PYTHONPATH`; loader and flat policy initialization pass. |
| Passing all `hi_pusht.yaml` `world` keys to current PushT caused `PushT.__init__()` to reject `history_size`. | Delete `world.history_size` and `world.frame_skip` with Hydra overrides for this dependency version. PushT reset and policy attachment pass. |
| Synthetic Hi-LeWM forward with a batch of one initially hit BatchNorm’s training-mode requirement. | Call `.eval()` as the evaluator does; the synthetic shape check passes. |
| Venv under `/mnt/c` caused a Python import process to stall in filesystem I/O. | Recreated the canonical venv on WSL’s native filesystem; validation then completed. The first workspace-mounted venv remains noncanonical and should not be used. |

## Not tested / remaining blockers

- No full evaluation or scientific experiment; no task-success result.
- No PushT evaluation has been run. The official dataset was downloaded and its HDF5 container was opened for the minimal path check recorded below.
- No Hi-LeWM learned high-level checkpoint downloaded, loaded, or trained.
- No NVIDIA/CUDA path exists on this host. CPU evaluation speed over the full study matrix is unknown.
- The Hi evaluation’s ordinary documented command needs the `PYTHONPATH` and Hydra field-deletion overrides above with this package version.
- Windows/WSL rendering was verified only through construction/reset; no video rendering or long episode was run.
- Exact package freeze is provided, but PyPI source/wheel hashes are not recorded for every distribution. The uv archive and downloaded PushT weights have recorded SHA256 values.

## Shared PushT dataset cache

The official `pusht_expert_train.h5.zst` from [`quentinll/lewm-pusht`](https://huggingface.co/datasets/quentinll/lewm-pusht/blob/main/pusht_expert_train.h5.zst) is stored once, with its extracted HDF5, on the WSL-native filesystem at `/home/bhavana/.stable_worldmodel`. This cache is shared by LeWorldModel, Flat RH, and Hi-LeWM evaluations when they use the same `STABLEWM_HOME`.

After extraction, WSL reported 949,814,431,744 bytes free on the filesystem containing this cache (about 885 GiB).

| Artifact | Location | Size | SHA-256 / verification |
|---|---|---:|---|
| Official compressed archive | `/home/bhavana/.stable_worldmodel/pusht_expert_train.h5.zst` | 13,136,247,974 bytes | `7cfbd6d90fa2f27876379a5ff169715a36ed82edbda64f9e5b5bfa34d212f318`; matches the official published checksum. |
| Extracted dataset | `/home/bhavana/.stable_worldmodel/pusht_expert_train.h5` | 46,300,921,856 bytes | Local SHA-256 `b6ebd9ac94bbe9e383f6e7a9cd92d74e9aa665ea57b758ed3717b0ee7df8d4fb`. No official extracted-file checksum was listed. `h5py` opened the file successfully; root keys: `action, ep_len, ep_offset, episode_idx, pixels, proprio, state, step_idx`. |
| Base LeWM object checkpoint | `/home/bhavana/.stable_worldmodel/pusht/lewm_object.ckpt` | 72,345,781 bytes | `91825ebcc183f6119deae85d0c98d82b68b1c0f151769c9d2ba8544b3e2796a0`; matches the previously validated checkpoint. |

`STABLEWM_HOME=/home/bhavana/.stable_worldmodel` is added to the WSL user's `.profile` and `.bashrc`. To make a command independent of shell startup behavior, prefix it explicitly:

```bash
export STABLEWM_HOME=/home/bhavana/.stable_worldmodel
```

Minimal validation used the prepared Python 3.10.21 environment and passed `STABLEWM_HOME` directly to the process. The expected evaluator dataset path exists, the HDF5 container opens, and the checkpoint is present at the normal `pusht/lewm_object.ckpt` cache location used by the `pusht/lewm` policy mapping. The evaluator was not invoked and no episode, trajectory, action, or task metric was produced. The existing runtime checkpoint in `.research_env/checkpoints` was copied into the shared cache; dataset archive and extraction exist only in the shared cache (no C: dataset duplicate).

## Isolated stable-worldmodel 0.1.0 candidate

Created a separate WSL-native candidate venv at `/home/bhavana/lossfunk_autoresearch/.venv-swm010`; the current `.venv-linux` was not modified. Candidate `STABLEWM_HOME` is `/home/bhavana/lossfunk_autoresearch/.candidate-cache-swm010`. Its checkpoint cache contains symlinks to the unchanged official `weights.pt` and `config.json`; its dataset cache contains a symlink to the existing shared HDF5. No checkpoint bytes or dataset bytes were copied or changed.

The candidate uses Python 3.10.21, `stable-worldmodel==0.1.0`, `stable-pretraining==0.1.6`, `torch==2.14.1+cpu`, `torchvision==0.29.1+cpu`, `gymnasium==1.3.0`, `hydra-core==1.3.7`, and `numpy==2.2.6`. The complete candidate package freeze is [ENVIRONMENT.candidate-swm010.lock.txt](ENVIRONMENT.candidate-swm010.lock.txt); its preflight code and reports are `research/instrumentation/preflight_swm010.py` and `research/results/swm010_compatibility_preflight_attempt{1,2}.json`.

| Software-only check | Result |
|---|---|
| Imports and exact loader API | Passed. With only `import stable_worldmodel`, `stable_worldmodel.wm.utils.load_pretrained` is available. |
| Official checkpoint | Passed. Loader instantiated `stable_worldmodel.wm.lewm.lewm.LeWM`; strict state loading succeeded for all 303 tensors, all tensor shapes match, parameter count 18,034,478. Official weights SHA-256 remains `48938400ae3464c9680731287f583a9cb516f55a8ec64ea13a91be47fb15b607`; official config was linked unchanged. |
| Existing PushT config and world | Passed. Checked-in Hydra config composed without overrides beyond `policy=pusht/lewm`; a 50-slot `swm/PushT-v1` vector world constructed. No environment step or reset episode was run. |
| Evaluator dataset and preprocessing setup | Passed. The official HDF5 was read through a candidate-cache symlink; evaluator cached `action`, `proprio`, and `state`, then fitted the configured scalers. No episode was evaluated. |
| Existing CEM planner | Could not initialize with the checked-in `device: cuda`: PyTorch raised `Cannot get CUDA generator without ATen_cuda library`. |
| Evaluator's hard-coded model device | Could not proceed: `eval.py` calls `model.to("cuda")`, which raised `Torch not compiled with CUDA enabled`. Consequently the evaluator did not construct its configured solver/policy and could not reach the pre-episode boundary. |

Installation history: resolving `stable-worldmodel[train]==0.1.0` together with the requested `stable-pretraining==0.1.6` is unsatisfiable according to package metadata because the train extra requires `stable-pretraining>=0.1.7`. The broad `[env]` extra also attempted to build unrelated `box2d-py` and failed because `swig` was unavailable on the build path. For this requested inference-only check, the candidate was therefore installed with bare `stable-worldmodel==0.1.0`, exact `stable-pretraining==0.1.6`, and the individually pinned evaluation/PushT dependencies in its lock. A first preflight harness attempt had two setup mistakes (it queried `World.action_space`, which is exposed through the vector env, and linked the dataset at the wrong cache level); both failures are preserved in the attempt-1 report. The corrected attempt-2 report records the passing checkpoint, environment, and dataset checks above.

This candidate establishes API and official-weight compatibility, but **does not establish a defensible end-to-end baseline runtime on this host**. The optional training-extra dependency declaration conflicts with the requested stable-pretraining version, and the repository's configured CEM/evaluator path requires CUDA while this WSL host has CPU-only PyTorch and no CUDA GPU. No device substitution or planner override was made. No scientific experiment, model inference, policy action, CEM evaluation, or episode was run.

## Remote GPU requirements and storage

### Host limitation and CUDA evidence

Read-only Windows inspection found one display adapter, `AMD Radeon(TM) 610M`, driver `32.0.21030.13004`; no NVIDIA adapter or NVIDIA driver was present, and Windows `nvidia-smi` was not available. WSL exposes `/dev/dxg`, but has no `nvidia-smi` binary or `/usr/lib/wsl/lib/nvidia-smi`. Candidate PyTorch reports `2.14.1+cpu`, `torch.cuda.is_available() == False`, `torch.version.cuda is None`, and zero CUDA devices. The Windows driver version above is the AMD display-driver version, not an NVIDIA/CUDA driver version. No CUDA-supported driver version applies. Installing a CUDA-enabled PyTorch wheel locally would not add NVIDIA hardware.

### Inference memory estimate (not benchmarked)

The official checkpoint contains 18,034,478 parameters. At FP32, parameter storage is about 68.8 MiB; the official `weights.pt` file is 72,290,721 bytes (about 69 MiB). LeWM uses a ViT-tiny encoder at 224x224 and a six-layer predictor with 192-dimensional embeddings. The checked-in LeWM CEM has `batch_size=1`, `num_samples=300`, and planned action length 25 (5 planner steps x 5 actions per block). With two float32 action coordinates, one raw baseline candidate population is only 300 x 25 x 2 x 4 = 60,000 bytes, excluding the model's intermediate tensors. The solver batches the 50 vectorized environments one at a time, so the config does not imply 50 simultaneous model/CEM populations on GPU.

Hi-LeWM's checked-in hierarchical config uses the same model object for its high and low solvers (rather than loading a second world-model copy); the solver populations are 900 and 600 respectively, with CEM batch size 1. Raw action/candidate tensors are small relative to framework, CUDA-context, image-encoder and predictor activations. **Peak VRAM was not measured**, because no CUDA device is available. Based on parameter size and sequential batch configuration, use **4 GB VRAM as a practical lower bound for a one-episode smoke and the existing baseline**, and **8 GB recommended for the full horizon/compute study** to leave room for longer candidate sequences, CUDA runtime allocations, and implementation variation. These are conservative engineering estimates, not repository-declared minima or validated guarantees; 2 GB might fit some inference shapes but is not a dependable target. Full-study throughput also depends on GPU speed and is not inferred from VRAM capacity.

| Workload | Smallest practical planning target | Confidence/qualification |
|---|---|---|
| Software-only smoke on the actual CUDA evaluator | One NVIDIA GPU, 4 GB VRAM | Estimated from one model and configured CEM batch; no peak allocation measured. |
| Checked-in 50-environment baseline reproduction | One NVIDIA GPU, 4 GB VRAM; 8 GB preferred | `batch_size=1` serializes model/CEM candidate batches across vector environments. This does not reduce the required 50 evaluation trajectories. |
| Full study with horizon sweeps and hierarchy | One NVIDIA GPU, 8 GB VRAM recommended; 4 GB may fit but is unverified | Larger candidate populations and longer planned sequences raise memory and runtime. No multi-GPU code path is required by the inspected evaluator. |

A **single NVIDIA GPU is sufficient by code structure**: each evaluator constructs one LeWM instance, and hierarchical mode passes that same model to both solvers. Runs can be performed serially. The repositories do not specify multi-GPU inference or distributed evaluation requirements.

### CUDA/software compatibility

The Hi-LeWM repository's documented `environment-gpu.yml` requests Python 3.10, CUDA Toolkit 12.1, and cuDNN 8.9. This is the only explicit CUDA version information found. The standalone LeWM repository does not pin a CUDA toolkit, NVIDIA driver minimum, or CUDA-specific PyTorch wheel; its CEM and evaluator configs simply request `device: "cuda"`. Therefore CUDA 12.1/cuDNN 8.9 is a repository-documented setup, not a validated requirement for the LeWM evaluation path. A remote machine must have a working NVIDIA driver/runtime combination compatible with its installed CUDA-enabled PyTorch; exact minimum driver and wheel versions remain to be established during remote software-only validation.

The Python 3.10.21 and model/evaluator dependency pins can be carried to Linux, but the current candidate lock cannot be installed unchanged because it pins CPU-only PyTorch. A separate GPU lock must pin the CUDA PyTorch/torchvision artifacts and all package versions. There is also a metadata conflict: `stable-worldmodel[train]==0.1.0` requires `stable-pretraining>=0.1.7`, whereas the checkpoint-compatible pair tested here uses `stable-pretraining==0.1.6`; the successful candidate omitted the train extra and installed inference/evaluation dependencies explicitly. Do not call a remote environment reproduced until the exact GPU lock, CUDA/driver versions, checkpoint load, PushT construction, and planner initialization are recorded and pass software-only checks.

### Remote data and disk

The existing dataset is on the local WSL filesystem at `/home/bhavana/.stable_worldmodel/pusht_expert_train.h5`; the official compressed archive is alongside it. A remote Linux machine cannot use that local path automatically. Reuse the identical data by transferring the extracted file (46,300,921,856 bytes, about 43.12 GiB) or redownloading the official archive (13,136,247,974 bytes, about 12.23 GiB) and extracting it. Retaining both archive and extracted dataset requires about 59.44 GB (55.36 GiB), plus the checkpoint (~69 MiB), environment, temporary files, and outputs. **At least 60 GB free is the data-only staging floor; allocate 80 GB or more usable disk** for environment/install temporaries and run artifacts. Configure remote `STABLEWM_HOME` to a persistent cache so all conditions/horizons share a single dataset copy. Verify the archive SHA-256 against the recorded official checksum or, for a transfer, verify the extracted file against the recorded local SHA-256.

No provider is selected in this assessment. No remote machine was provisioned, no files were transferred, and no scientific experiment was run.
