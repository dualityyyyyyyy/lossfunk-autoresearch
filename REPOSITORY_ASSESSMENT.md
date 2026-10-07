# Repository Assessment: LeWorldModel and Hi-LeWM

## Scope and evidence labels

This is a source, documentation, and repository-history inspection only. No training, policy evaluation, scientific diagnostic, test suite, dry run, dependency installation, dataset download, checkpoint download, or result-producing command was run. The code is present, but runtime behavior remains **not tested**.

- **Verified from source** means the checked-out source/config contains the described implementation or setting; it does not mean it executed successfully.
- **Documented by repository** means its README/config/docs say so.
- **Inferred from source** means the interpretation follows the visible code/config but depends in part on a library whose implementation was not in the checkout.
- **Not tested** means no runtime verification was attempted.

## Repositories and commits

| Repository | URL | Checked-out commit | Notes |
|---|---|---|---|
| LeWorldModel (`le-wm`) | https://github.com/lucas-maes/le-wm.git | `8edfeb336732b5f3ce7b8b210d0ba370a09e2cac` | Standalone repository checked out at this commit. |
| Hi-LeWM (`Hi-LeWM`) | https://github.com/NiccoloCase/Hi-LeWM.git | `4bb21a2888e8f22b8d084762c80361e398968775` | Includes a LeWorldModel git submodule. |
| Hi-LeWM pinned LeWorldModel submodule | https://github.com/lucas-maes/le-wm.git | `83f97d72ad067855bc89a1b74b4aff11d4dfdf0c` | Submodule path `Hi-LeWM/third_party/lewm`; also recorded in `Hi-LeWM/BASELINE_LOCK.md`. |

The Hi-LeWM README gives its clone URL as `NiccoloCase/h-le-wm.git`, but the requested repository URL above was the one cloned and assessed. Do not silently substitute these URLs when reproducing.

## Executive assessment

**Cleanest starting point:** use Hi-LeWM’s evaluation harness and its exact pinned LeWorldModel baseline, because the checked-in `hi_eval.py` already has `planning.mode=flat`, `hierarchical`, and `hierarchical_staged`. This avoids writing a new acting policy merely to add flat receding-horizon planning. Its flat mode delegates to the same `stable_worldmodel.policy.WorldModelPolicy` API used by LeWorldModel’s evaluation script. This conclusion is verified from source; the existing flat path has also completed a one-episode instrumentation smoke, which is not a scientific baseline result.

LeWorldModel is the cleaner reference for the original model and baseline invocation. Its repository intentionally delegates environment, CEM, and policy mechanics to the external `stable-worldmodel` package. The exact package version is not pinned in either repository’s dependency declaration. The checked-out LeWM evaluator calls `stable_worldmodel.wm.utils.load_pretrained`, which is absent from the installed `stable-worldmodel==0.0.6` wheel. In an isolated candidate environment, `stable-worldmodel==0.1.0` with `stable-pretraining==0.1.6` provides this API, strictly loads the unchanged official checkpoint with all 303 tensors and shapes matching, and constructs the configured PushT world and evaluator preprocessing. The configured evaluator/planner still cannot initialize on this CPU-only host: evaluator code transfers the model to CUDA, and CEM creates a CUDA generator. In addition, the `stable-worldmodel[train]` extra for 0.1.0 requires `stable-pretraining>=0.1.7`, so the tested exact pair required omitting that extra and installing the inference/evaluation dependencies explicitly. See `ENVIRONMENT.md` and the preflight reports. End-to-end baseline runtime remains unvalidated here.

## LeWorldModel: PushT baseline and code map

### Cleanest reproducible PushT baseline

The repository-documented baseline is a pretrained PushT LeWM checkpoint evaluated by `le-wm/eval.py` with `le-wm/config/eval/pusht.yaml` and the CEM solver config. The README command is:

```bash
python eval.py --config-name=pusht.yaml policy=pusht/lewm
```

The current PushT config specifies `seed: 42`, 50 evaluations, dataset `pusht_expert_train`, `goal_offset_steps: 25`, `eval_budget: 50`, CEM `num_samples: 300`, `n_steps: 30`, `topk: 30`, `horizon: 5`, `receding_horizon: 5`, and `action_block: 5`. It leaves the policy as `random` until overridden. These values are **verified from source/config**; they are not evidence that this exact run is reproducible or has been executed.

For a controlled experiment, freeze this config and checkpoint, explicitly set every seed and planning field, capture the resolved Hydra config, and decide whether to retain the repository’s dataset-start/goal setup or implement the study’s horizon sweep. The repo defaults alone do not implement the required study design.

### Relevant files and responsibilities

| Responsibility | Files and source-visible symbols | Evidence/notes |
|---|---|---|
| Model definition / training-time objective | `le-wm/jepa.py`: `JEPA.encode`, `JEPA.predict`, `JEPA.rollout`, `JEPA.get_cost`; `le-wm/module.py`: `ARPredictor`, `Embedder`, `MLP`, transformer blocks; `le-wm/train.py`: `lejepa_forward`, `run` | **Verified from source.** `JEPA.rollout` encodes initial pixels and autoregressively predicts embeddings under candidate actions; `get_cost` compares terminal predicted embedding with encoded goal embedding. |
| Evaluation-time model loading | `le-wm/eval.py`: `run`; calls `swm.wm.utils.load_pretrained(cfg.policy)` in this revision | **Verified from source.** It moves the model to literal `cuda`, sets eval/no-grad behavior, and instantiates solver/policy from configs. |
| Planning | `le-wm/eval.py`: Hydra-instantiates `cfg.solver`, constructs `swm.PlanConfig`, then `swm.policy.WorldModelPolicy`; `le-wm/config/eval/solver/cem.yaml`: `stable_worldmodel.solver.CEMSolver` | The CEM solver and policy implementation are **external**, not defined in this repo. Solver fields visible here: `batch_size`, `num_samples`, `var_scale`, `n_steps`, `topk`, `device`, `seed`. |
| Action execution / environment loop | `le-wm/eval.py`: `swm.World`, `world.set_policy`, `world.evaluate`; policy is external `WorldModelPolicy` | **Verified from source** at call sites. Exact environment stepping and action buffering are owned by external `stable-worldmodel`; not locally inspectable here. |
| Data and image preprocessing | `le-wm/eval.py`: `get_dataset`, `img_transform`, dataset-start selection; `le-wm/config/eval/pusht.yaml` | Uses PushT HDF5 data and state-setting callables in the config. |

### Flat open-loop and receding-horizon planning

- **Flat open-loop (A):** the baseline config sets `horizon: 5` and `receding_horizon: 5`. In Hi-LeWM’s own policy implementation, a low-level plan contributes `receding_horizon * action_block` actions to its execution buffer. Applying that same documented/configured convention to the external flat `WorldModelPolicy`, setting `receding_horizon == horizon` means execute the full planned horizon before replanning. This specific flat-policy interpretation is **inferred from source/config**, because `WorldModelPolicy` is in the unpinned external package and was not inspected or run.
- **Flat receding-horizon (B):** a receding-horizon field already exists in the LeWM evaluation config and is also passed into `WorldModelPolicy` via `swm.PlanConfig`. Hi-LeWM’s evaluator already exposes a `flat` mode that routes through that same API and a flat `plan_config`. Therefore no new planner algorithm appears necessary. **Smallest change:** use a frozen config with total lookahead `horizon=K`, `receding_horizon=r` where `r<K`, and fixed `action_block`; evaluate the new observation and call the same policy again after the `r * action_block` executed actions. If the external policy does not honor the field as expected, the minimal fallback is to wrap/reuse `WorldModelPolicy` and control its plan buffer/replan boundary; inspect the installed package before implementing this fallback.
- **Can B reuse A’s components?** Yes at the source/config interface: same baseline model/checkpoint, CEM solver class/config, environment, preprocessing, and `WorldModelPolicy`; vary only horizon/replanning fields to begin. Whether those settings alone make execution exactly as intended is **not tested**. The comparison still needs measured work because changing `r` changes planner-call count and environment-to-planner feedback.

### Inference, CEM, and execution controls

| Quantity | LeWM control / visibility | Caveat |
|---|---|---|
| Model rollout lookahead | `plan_config.horizon` | Unit is planner steps, not necessarily environment steps. |
| Number of actions per planner step | `plan_config.action_block` | PushT config comment calls this frameskip; solver sees grouped actions. |
| Execute/replan interval | `plan_config.receding_horizon` | Exact `WorldModelPolicy` behavior is external and should be inspected/version-pinned. Hi policy code uses `receding_horizon * action_block`. |
| CEM population | `solver.num_samples` | Current LeWM default 300. |
| CEM iterations | `solver.n_steps` | Current LeWM default 30. |
| CEM elite count | `solver.topk` | Current LeWM default 30. |
| CEM variance scale | `solver.var_scale` | Current default 1.0. |
| Planner seed/device | `solver.seed`, `solver.device` | Config default seed interpolates evaluation seed; device defaults to CUDA. |
| Task start/goal horizon and budget | `eval.goal_offset_steps`, `eval.eval_budget`, selected dataset rows | Current defaults 25 and 50 respectively. |
| Model forward passes/evaluations | Not explicitly counted by the LeWM script | Add counters/hooks around model rollout/predict and solver objective calls; report actual counts. |
| Wall-clock inference | Total evaluation duration is appended by `eval.py` | This includes more than pure inference; isolate per-planning and per-model-call time with explicit instrumentation and device synchronization. |
| Parameters | `model.state_dict()`/parameter enumeration | Not reported by evaluator; count and state counting convention. |

## Hi-LeWM: hierarchy, subgoals, and reuse

### Hierarchical implementation and subgoal flow

| Stage | Source files / symbols | What source shows |
|---|---|---|
| Training waypoint selection | `Hi-LeWM/hi_waypoint_sampling.py`: `sample_waypoints` and strategy helpers; `Hi-LeWM/hi_train.py`: `hi_lejepa_forward`, `build_p2_frozen_waypoint_collate` | Samples ordered observation indices within a data sequence using configured strategies (`random_middle`, `random_sorted`, `fixed_stride`); action sequences between adjacent selected observations become action chunks. |
| Macro-action representation | `Hi-LeWM/hi_module.py`: `LatentActionEncoder`; `Hi-LeWM/hi_vq.py`: optional VQ implementation | A transformer encoder maps each variable-length primitive-action chunk, with mask, into a learned continuous or quantized macro-action latent. The main train config selects continuous. |
| High-level dynamics | `Hi-LeWM/hi_jepa.py`: `HiJEPA.encode_macro_actions`, `predict_high`, `rollout_high`; `Hi-LeWM/hi_train.py`: `hi_lejepa_forward` | High predictor consumes waypoint context latent plus encoded macro-action and predicts the next observation waypoint latent. Training target is encoded future waypoint image, with high-level prediction loss. |
| High-level CEM/subgoal proposal | `Hi-LeWM/hi_policy.py`: `HierarchicalWorldModelPolicy._plan_high`; `Hi-LeWM/hi_eval.py`: `build_policy` | High-level CEM samples macro-action sequences. The selected action prefix is passed through `rollout_high`; the first predicted latent becomes `_z_subgoal`. |
| Low-level planner | `Hi-LeWM/hi_policy.py`: `_plan_low`, `get_action`; `hi_eval.py`: low solver construction | Low-level CEM searches primitive/grouped actions with `planner_level="low"`, `z_hist`, `a_hist`, and `z_subgoal`; the output is buffered and sent to the environment. When the buffer empties it replans; high-level replanning occurs on `replan_interval`. |
| Stage-wise variant | `StagedHierarchicalWorldModelPolicy` in `hi_policy.py` | Plans the high-level sequence once, then moves through its predicted latent targets by stage duration; low-level planning remains closed-loop. Distinct from periodically regenerated hierarchy. |

Subgoals are **latent observation embeddings** (the model’s encoded waypoint states), not decoded images and not ground-truth/oracle state targets in the acting policy. Macro-actions are the high-level CEM search variables, while predicted latent states are consumed as low-level targets. This distinction is **verified from source**. The README documents the same design. Decoder probes are analysis tools and are not part of policy action generation.

### Reuse and checkpoint compatibility

- `Hi-LeWM/hi_train.py` loads an object-format pretrained LeWM model when `pretrained_low_level.enabled` is true. It reuses its encoder, low predictor, action encoder, projector, and low prediction projection; the defaults freeze these low-level pieces and train the high-level predictor, macro-action encoder, and related high-level modules. **Verified from source/config.**
- `Hi-LeWM/hi_eval.py` loads an object checkpoint through `swm.policy.AutoCostModel(cfg.policy)` and chooses flat or hierarchical policy construction from config. The flat branch uses `WorldModelPolicy`; the hierarchical branch builds separate high/low solvers over the loaded model. **Verified from source.**
- Consequently, the same *base LeWM PushT object checkpoint* can anchor A and B and initialize the low-level modules for C. C still needs separately trained/saved high-level components. The same raw LeWM checkpoint is not itself a trained hierarchical policy. This is **inferred from loader and model assembly**, not runtime-verified.
- Hi-LeWM’s README points to an archive at DOI `10.5281/zenodo.21353240` and its configs refer to `pusht/hi_lewm`; no checkpoint is checked into Git. Exact public artifact filenames/content and the availability of a matching Hi-LeWM PushT object checkpoint were **not verified** in this inspection. Do not assume config policy names imply the artifact is locally present.

## Checkpoints and data availability

### LeWorldModel

- **Documented by repository:** Hugging Face model repository `quentinll/lewm-pusht`, with `weights.pt` (state dict) and `config.json`; README conversion code constructs a `JEPA` and saves an object checkpoint expected at `$STABLEWM_HOME/pusht/lewm_object.ckpt`.
- **Externally inspected for availability:** the Hugging Face model page identifies it as the official pretrained PushT model and lists `config.json` and `weights.pt`. The `weights.pt` page reports a 72.3 MB remote file and SHA256 `48938400ae3464c9680731287f583a9cb516f55a8ec64ea13a91be47fb15b607`. This was metadata/page inspection only; no file was downloaded. Availability can change. Source: [LeWM PushT checkpoint files](https://huggingface.co/quentinll/lewm-pusht/tree/main) and [weights.pt metadata](https://huggingface.co/quentinll/lewm-pusht/blob/main/weights.pt).
- **Documented alternative:** checkpoint tar archives contain `<name>_object.ckpt` and `<name>_weight.ckpt`; the README lists a Google Drive checkpoint suite.
- **Loading:** LeWM README describes `swm.policy.AutoCostModel('pusht/lewm')`. The standalone `le-wm/eval.py` at checked-out commit uses `swm.wm.utils.load_pretrained(cfg.policy)` instead. The Hi-LeWM pinned wrapper’s `third_party/lewm/eval.py` uses `AutoCostModel`. This loader difference is material: use the loader paired with the chosen repository/version and verify format compatibility; do not assume loaders are interchangeable.

### Hi-LeWM

- **Documented by repository:** checkpoints are linked to DOI `10.5281/zenodo.21353240`; hierarchical training writes an object dump under the stable-worldmodel cache using `output_model_name` (default/configured model names include `hi_lewm`).
- **Not established:** a PushT checkpoint filename, exact archive contents, whether the artifact is downloadable without restrictions, and whether it matches this checkout’s exact architecture/config. No checkpoint was downloaded or loaded.
- **PushT data:** both evaluation configs name `pusht_expert_train`; the README says data should be placed beneath `STABLEWM_HOME` and offers `scripts/setup_datasets.sh`. Dataset files were not downloaded.

## Installation, dependency requirements, and versioning

### Repository-documented commands

LeWorldModel README:

```bash
uv venv --python=3.10
source .venv/bin/activate
uv pip install 'stable-worldmodel[train,env]'
```

Hi-LeWM README (recursive clone is needed for the submodule; replace with the exact requested URL when provisioning this checkout):

```bash
git clone --recursive https://github.com/NiccoloCase/Hi-LeWM.git
cd Hi-LeWM
conda env create -f environment.yml
conda activate lewm
```

GPU environment documented by Hi-LeWM:

```bash
conda env create -f environment-gpu.yml
conda activate lewm-gpu
```

Hi-LeWM documents a minimal `uv` alternative:

```bash
uv venv --python=3.10
source .venv/bin/activate
uv pip install 'stable-worldmodel[train,env]' pytest
```

The documented dataset helper uses Bash `source`:

```bash
source scripts/setup_datasets.sh --datasets pusht
```

These are **documented install commands, not tested commands**. The `environment.yml` and `environment-gpu.yml` both declare Python 3.10 and install `stable-worldmodel[train,env]`; the first also lists `pytest`. The GPU file also asks Conda for CUDA Toolkit 12.1 and cuDNN 8.9. No explicit versions are pinned for the central Python dependencies, and there is no checked-in fully resolved cross-platform lock file. Transitive requirements include PyTorch, Lightning, Hydra/OmegaConf, stable-pretraining, stable-worldmodel, image/data and PushT environment packages, but exact compatible versions are not frozen here.

**Python:** Python 3.10 is the explicit documented requirement for both repositories. No evidence was found that Python 3.14 (the current workspace interpreter) is supported by these dependency sets. It was not tested.

**Minimum dependency set:** for the provided harness, the repository claims `stable-worldmodel[train,env]` is the principal install, plus `pytest` for Hi-LeWM tests. Actual transitive dependency requirements and installation success are **not tested**. LeWM’s model implementation imports PyTorch and einops directly and its scripts additionally use Hydra, NumPy, Lightning, stable-pretraining, torchvision, scikit-learn, and stable-worldmodel; rely on declared package extras only after pinning and resolving them.

### GPU/CPU requirements

- The LeWM baseline evaluator calls `model.to("cuda")`; its CEM config defaults to `device: "cuda"`. Thus the supplied baseline command expects a CUDA-capable PyTorch environment **as written**. No minimum GPU model or VRAM is documented in these files.
- Hi-LeWM’s PushT config likewise defaults both high and low CEM solvers to CUDA. `hi_eval.py` selects its loaded model device from solver config and checks high/low device agreement. Source therefore permits a CPU override in principle, but performance and compatibility are **not tested**. The `environment.yml` is named CPU/dev yet the evaluation config remains CUDA by default; explicitly override solver devices for a CPU attempt.
- Training configs set GPU accelerator and bf16 by default; documented training therefore expects compatible GPU hardware. No minimum GPU/VRAM guarantee is given.
- Workspace Python package inspection found PyTorch, stable-worldmodel, stable-pretraining, Gymnasium, and MuJoCo are not installed. This is an environment inventory only, not a repository run.

### Windows, Linux/WSL, and Colab

- **Direct Windows support:** not documented and not tested. The repo installation and dataset commands use Bash activation/`source`; LeWM eval sets `MUJOCO_GL=egl`; configs default to CUDA; and the PushT environment stack includes native simulation/rendering dependencies. This combination is a likely direct-Windows friction point, but the source alone does not prove Windows cannot run it.
- **Recommended execution environment:** Linux, including WSL2 Ubuntu or Google Colab. Use Python 3.10 and the repository’s documented Linux commands. WSL2 with a supported NVIDIA CUDA setup is the natural route for local GPU evaluation; Colab is suitable only after matching the pinned Python/PyTorch/CUDA stack and arranging persistent dataset/checkpoint storage.
- **What must run under Linux/WSL/Colab:** no source file formally says “must,” but training/evaluation commands, EGL rendering, native PushT/MuJoCo dependencies, and Bash dataset setup should be treated as Linux-targeted until a clean Windows installation is explicitly tested. This is an operational recommendation/inference, not a verified prohibition.

## Dependency conflicts and reproducibility blockers

1. Both projects request Python 3.10 and the `stable-worldmodel[train,env]` extra, so their documented requirements show no direct version conflict.
2. Neither repository pins exact versions for stable-worldmodel, stable-pretraining, PyTorch, torchvision, Lightning, Hydra, or NumPy. Independently installing the extra at different dates can produce drift. A lockfile/container is required before comparing outputs.
3. Hi-LeWM pins LeWM source as a submodule, but the pip-installed stable-worldmodel package remains floating. The checked-out standalone LeWM commit (`8edfeb…`) differs from Hi-LeWM’s baseline submodule commit (`83f97d…`); the latter is explicitly locked by `BASELINE_LOCK.md`. Use the pinned submodule behavior for a clean comparison, not an unqualified latest package/API assumption.
4. Checkpoint deserialization is Python-object based for the documented evaluation path. This creates coupling to module/class import paths and package versions. Hi-LeWM adds a dynamic baseline adapter for legacy pickle module names and a CPU map-location wrapper, but these compatibility paths are source evidence only and remain untested.
5. Hi-LeWM lists `pytest` in its CPU/dev environment; its GPU environment omits it. This is a tooling difference rather than an identified runtime dependency conflict.
6. No dependency resolution/install was attempted, so actual package conflicts, ABI failures, or Windows build blockers remain unknown.

## Evaluation commands and relevant diagnostics

### Repository-documented PushT commands (not run)

LeWorldModel:

```bash
python train.py data=pusht
python eval.py --config-name=pusht.yaml policy=pusht/lewm
```

Hi-LeWM flat baseline wrapper:

```bash
python train.py data=pusht
python eval.py --config-name=pusht policy=pusht/lewm
```

Hi-LeWM standard hierarchical PushT evaluation:

```bash
python hi_eval.py --config-name=hi_pusht policy=pusht/hi_lewm
```

Hi-LeWM empirical-macro hierarchical variant:

```bash
python hi_eval.py --config-name=hi_pusht policy=pusht/hi_lewm \
  planning.high.empirical_macro.enabled=true \
  planning.high.empirical_macro.num_sequences=4096
```

README-documented wrapper dry runs (not run; not scientific evaluations):

```bash
LEWM_WRAPPER_DRY_RUN=1 python train.py data=pusht
LEWM_WRAPPER_DRY_RUN=1 python eval.py --config-name=pusht
LEWM_WRAPPER_DRY_RUN=1 python hi_eval.py --config-name=hi_pusht
```

The dry-run commands only print wrapper commands when the environment variable is set; they do not validate dependency availability, checkpoint loading, planner behavior, or environment execution.

### Existing evaluation and diagnostic code relevant to PushT

| File(s) | Purpose and status |
|---|---|
| `le-wm/eval.py`; `le-wm/config/eval/pusht.yaml`; `le-wm/config/eval/solver/cem.yaml` | LeWM PushT evaluator, evaluation sampling/config, CEM solver target. |
| `Hi-LeWM/eval.py`; `Hi-LeWM/third_party/lewm/eval.py` | Thin wrappers that delegate baseline execution through `baseline_adapter.py` to the pinned submodule. |
| `Hi-LeWM/original_eval_with_manifest.py` | Pinned-baseline evaluation wrapper with per-episode manifest handling; inspect its CLI before choosing it. No README invocation is provided. |
| `Hi-LeWM/hi_eval.py`; `Hi-LeWM/config/eval/hi_pusht.yaml` | Flat, periodically hierarchical, and staged hierarchical evaluation modes, selected from config. |
| `Hi-LeWM/scripts/run_hi_diagnostic.py`; `scripts/hi_diagnostics.py` | Teacher-forced/open-loop/high-level planner latent-prediction diagnostic tooling. |
| `Hi-LeWM/scripts/run_hi_acting_diagnostic.py`; `scripts/hi_acting_diagnostics.py` | Diagnostic for whether selected subgoals can be reached using the low-level planner. |
| `Hi-LeWM/hi_decoder_probe.py`; `hi_train_decoder_probe.py`; `hi_decoder_probe_eval.py`; `scripts/run_decoder_probe_report.py`; decoder-probe rendering helpers | Trains/evaluates a diagnostic decoder and renders latent-waypoint panels. Not part of acting policy. |
| `Hi-LeWM/eval_determinism.py` | Process-level determinism setup/report used by Hi evaluation. |
| `Hi-LeWM/scripts/test_macro_action_manifold.py` | Macro-action support/manifold diagnostic. |
| `Hi-LeWM/tests/test_hi_planning.py`, `test_hi_eval_sampling.py`, `test_waypoint_sampling.py`, `test_hi_decoder_probe.py`, `test_hi_train_speedups.py` | Unit/smoke tests relevant to planning, sampling, probes, and training code. Not run. |
| `Hi-LeWM/scripts/render_hi_paper_diagnostics.py`, `render_hi_story_figures.py`, `render_hi_decoder_diagnostic_stories.py` | Render figures from diagnostic data; these are artifact-generation utilities, not independent evaluators. |

The README gives commands only for the main flat/hierarchical evaluations, wrapper dry runs, and `python -m pytest`; it does not document full CLI invocations for the individual diagnostics. Do not invent those as repository-provided commands. Inspect each script’s argparse/Hydra interface when selecting one. No script was executed.

## Proposed A/B/C mapping

All three conditions should use the same PushT environment, data-derived start/goal protocol, LeWM low-level model checkpoint, image/state/action preprocessing, evaluation seeds, task horizon definition, and fixed reporting/instrumentation. The research spec remains unchanged.

| Condition | Implementation mapping | Key setup |
|---|---|---|
| A. Flat open-loop | `WorldModelPolicy` through LeWM evaluator or Hi-LeWM `planning.mode=flat` | Flat CEM plans through the task horizon; execute the full plan before observing/replanning. Use full-horizon lookahead and matching action-block definition. |
| B. Flat receding horizon | Same flat policy, model, solver class, and environment as A | Shorter local lookahead and execute a fixed prefix, then observe and replan. Hi-LeWM already exposes flat config routing. No new policy is expected unless external library semantics differ from the visible API. |
| C. Hierarchical | `HierarchicalWorldModelPolicy` through `hi_eval.py` | Train learned high-level/macro-action model from PushT waypoints, use its predicted latent subgoals, and use the same LeWM low-level checkpoint/low-level planner family as B. Do not substitute oracle subgoals. |

For C, select and freeze one hierarchy variant (periodically replanned `hierarchical` or one-shot `hierarchical_staged`) before the study. The default `hierarchical` mode replans high level every `replan_interval`; its low policy also replans whenever its action buffer empties. These frequencies must be included in the comparison.

## Compute-matching strategy

Do not call conditions compute-matched from nominal CEM settings alone. Hi-LeWM’s default hierarchy has separate high/low solvers (defaults: high 900 samples × 20 iterations, low 600 × 30; `num_samples` is not automatically equal to actual model forward passes per iteration). LeWM flat CEM defaults to 300 × 30. Model rollouts can have different temporal lengths, and the high level predicts latent transitions while low level predicts primitive-action outcomes.

For every policy call and episode, instrument and preserve at least:

1. Actual solver objective/planner evaluations, split by level; actual CEM iterations and population sizes (including early termination if any).
2. LeWM low-level predictor forward calls and sequence lengths; high-level predictor forward calls and sequence lengths; image encoder calls; latent-action encoder calls. Count forward **examples/tokens** as well as call count when batch/time dimensions differ.
3. Number of selected actions executed, observations consumed, environment interactions, replans at each level, and each execution block size.
4. Total and per-call wall-clock inference time, with accelerator synchronization around measured GPU intervals; report device and precision.
5. Total, shared-low-level, and added-high-level parameter counts; exact checkpoint hashes.
6. Planning lookahead, task horizon, action block, high/low replanning frequency, CEM settings, and random seeds from resolved configs.

Then run a small **non-scientific instrumentation calibration** only after the experiment is authorized: verify counters on synthetic/configuration-level calls without scoring task success. Tune CEM populations/iterations (and, if justified, batch shape) so B and C land on a predeclared overlapping measured inference-computation budget. Keep configurations and report residual budget differences; also plot success versus measured compute per the frozen spec. Wall time is hardware-dependent and should be reported alongside, not used as the sole proxy for model computation. No such calibration was performed here.

## Minimal integration plan

1. Treat `Hi-LeWM/third_party/lewm` at `83f97d…` as the locked flat baseline; record all repository and dependency versions.
2. Resolve exact compatible dependencies and create a Linux/Python 3.10 lock/container. Verify (without scoring) import, checkpoint loading, environment reset/render, and one planner call; preserve outputs as implementation checks.
3. Use `hi_eval.py` flat mode as the shared evaluator entry point for A and B, if checkpoint loading is confirmed. Add/freeze separate A and B config groups only: task horizon, planning horizon, receding horizon, action block, evaluation seeds, and solver budget.
4. Use `hi_train.py` to train the learned high-level model for C on the same designated PushT training data while initializing/fixing the same LeWM low-level checkpoint. Store the exact final hierarchical object checkpoint and config. Treat this training as an experiment under the study protocol, not as setup.
5. Instrument the common low-level model, high-level model, CEM solver calls, environment steps, and timers before evaluations. Confirm instrumentation on non-scientific fixtures and save the raw check output.
6. Freeze horizon/seed/budget matrices and run A/B/C according to `RESEARCH_SPEC.md`; retain all failed runs and raw outputs.

No integration edits beyond this assessment were made.

## Expected risks and blockers

- Floating `stable-worldmodel` dependency means API, planner semantics, and behavior may drift; pin the package/source version before claiming reproducibility.
- Flat `WorldModelPolicy` internals are outside both the standalone LeWM repo and the Hi-LeWM submodule. Receding-horizon execution behavior is inferred from config/Hi policy analogues until that dependency is inspected.
- Checkpoint conversion/object-pickle loading may be sensitive to code version/module names. Hugging Face provides weights/config, while the evaluation API expects a serialized object in some paths.
- Hi-LeWM PushT trained artifact filenames and exact matching architecture were not confirmed. The DOI is documented, but no archive was downloaded.
- Dataset is required to reproduce the configured evaluation starts/goals and for high-level training; it was not downloaded. Data setup script is Bash-oriented.
- Hi hierarchy adds parameters and high-level inference; parameter count alone cannot match. Need measured forward counts and wall time, as required in the research specification.
- Standard CEM costs may change substantially with horizon and replanning rate. Budget matching can require tuning populations/iterations and careful reporting.
- Current PushT config defaults use only 50 episodes and one seed; those are repository defaults, not necessarily adequate for the frozen study.
- CUDA/EGL/native environment compatibility, CUDA version requirements, runtime speed, and direct Windows support remain unverified.

## Not tested / not concluded

Neither repository was installed or executed. No dry-run, test, diagnostic, training, evaluation, model inference, dataset/checkpoint download, or scientific result was produced. This assessment makes no scientific conclusion about which planner is more reliable.
