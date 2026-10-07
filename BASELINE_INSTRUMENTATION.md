# Baseline Runtime Instrumentation

## Scope and evidence labels

This is instrumentation preparation, not baseline reproduction. **Source verified** means checked in source or the installed dependency source. **Runtime verified** means observed in the one synthetic smoke decision described below. **Inferred** means directly deduced from source/config, but not measured in a real environment rollout. **Unknown** means this stage did not establish it.

No source file in either upstream repository was edited. The code added under `research/instrumentation/` uses reversible Python wrappers around policy, CEM, world-model, and environment-step call boundaries. Original call arguments and returned values are forwarded unchanged. It does not alter model weights, candidate tensors, objectives, solver settings, selected actions, or environment physics. Instrumentation has I/O and timing overhead; recorded durations are instrumented timings and must not be used as benchmark latency without separately characterizing that overhead.

## Checked-out execution path

The checked out revisions are LeWorldModel `8edfeb336732b5f3ce7b8b210d0ba370a09e2cac`, Hi-LeWM `4bb21a2888e8f22b8d084762c80361e398968775`, and the Hi-LeWM `third_party/lewm` submodule `83f97d72ad067855bc89a1b74b4aff11d4dfdf0c`.

The target runnable path is Hi-LeWM's `hi_eval.py` flat branch, which constructs the external `stable_worldmodel.policy.WorldModelPolicy`, and the same policy class in flat mode. Source inspection of installed `stable_worldmodel==0.0.6` verified:

1. `World.step()` calls `policy.get_action(self.infos)` and then `envs.step(actions)`.
2. `WorldModelPolicy.get_action()` preprocesses the observation mapping, calls its solver only when `_action_buffer` is empty, buffers the first `receding_horizon * action_block` actions, and returns one action per policy call.
3. `CEMSolver.solve()` draws `num_samples` candidates per environment per iteration and calls `model.get_cost(current_info, candidates)` once for each iteration/batch. It then updates the distribution and returns the final mean action sequence.
4. The pinned `JEPA.get_cost()` encodes the goal, calls `rollout()`, and computes candidate costs. `rollout()` encodes initial pixels, repeatedly calls `predict()` (including its final prediction), and stores predicted embeddings. The trace records each of these methods when present on the loaded model.

Hi-LeWM's `HierarchicalWorldModelPolicy.get_action()` similarly receives the per-decision info mapping. Its `_plan_high()` and `_plan_low()` construct different solver-info dictionaries (including `planner_level` and latent fields), invoke their solvers, and buffer low-level actions. The generic hooks below can wrap this policy too; the smoke test exercised flat policy only. The ordinary LeWM `eval.py` loads a model onto literal `cuda` and is not the selected CPU launch path; use Hi-LeWM's flat evaluator wrapper for the prepared environment.

## Added files

| File | Purpose |
|---|---|
| `research/instrumentation/trace_runtime.py` | Reversible wrappers for `get_action`, CEM `solve`, model `get_cost` and named model methods (`encode`, `predict`, `rollout`, high-level variants), plus `World.step`. Writes one JSON object per event. Trace creation is exclusive so an existing raw trace cannot be overwritten. |
| `research/instrumentation/run_traced_hi_eval.py` | Opt-in launcher around Hi-LeWM's unmodified `hi_eval.py`; captures resolved config, seed, Python/platform/device, checkpoint path/hash, and repository commits; installs hooks before evaluation. One real flat PushT smoke episode has now passed through it. A wrapper-only Hydra config-resolution fix is documented below. |
| `research/instrumentation/smoke_action_transition.py` | One-call fake policy/vector-environment check of action handoff and returned transition tracing; does not create PushT or call CEM/model inference. |
| `research/traces/action_transition_validation.jsonl` | Raw JSONL from that one-call instrumentation validation. |
| `research/instrumentation/smoke_trace.py` | One-decision synthetic-input smoke check using the real pinned LeWM checkpoint, `WorldModelPolicy`, and CEM. Does not instantiate or step a PushT environment. |
| `research/traces/instrumentation_smoke_topk2_1.jsonl` | Final raw trace after the wrapper refinements. |
| `research/traces/instrumentation_smoke_topk2_1_summary.json` | Small machine-readable summary for that final trace. |
| `research/traces/instrumentation_smoke_topk2.jsonl` | Earlier clean topk=2 smoke trace; retained. |
| `research/traces/instrumentation_smoke_summary.json` | Summary from the earlier clean topk=2 attempt. |
| `research/traces/instrumentation_smoke.jsonl` | Earlier smoke trace using `topk=1`; preserved, with its CEM `std()` degrees-of-freedom warning context recorded in the run log. |

## What each event measures

- `run_metadata`: immutable run context supplied by the launcher. The standard launcher records resolved config, seed, checkpoint path and SHA256 when found, Git commits, Python/platform, torch, CUDA availability, and model device.
- `policy_decision`: one call to `get_action`; records the original observation keys, shapes, dtypes, device and SHA256. Numeric values are included for arrays/tensors with at most 4096 elements; image-sized inputs are fingerprinted rather than copied into JSON. It also records action-buffer depth before/after and whether a solver call occurred during this policy call. This is the observation actually passed into the policy, before preprocessing.
- `cem_solve`: a solver call with the actual info dictionary handed to CEM. Small planner tensors (including `z_init`, `z_goal`, and `z_subgoal` when present) include their numeric values; large tensors are represented by shape/dtype/device. Solver fields, `PlanConfig` horizon/receding horizon/action block/history length, elapsed time, and returned action shape are recorded. The resolved run config records the configured seed (the external `CEMSolver` does not retain it as an instance attribute).
- `world_model_get_cost`: each successful scored candidate batch, its shape, returned cost tensor shape, number of candidate objects scored (`costs.numel()`), and elapsed wall time. For this CEM implementation the cost output must have shape `(batch, num_samples)`, so this counts candidate plans actually scored under the study's definition. Failed calls are logged as errors and do not increment the successful scored count.
- `candidates_generated_source_formula`: on completed CEM calls, `n_steps * num_samples * n_envs`. This is a source-verified count for the pinned CEM loop when it completes all iterations/batches. It is distinguished from the measured scored count; on a failed/incomplete solve it may overstate generated candidates and must not be treated as observed.
- `world_model_encode`, `world_model_predict`, `world_model_rollout`, `world_model_predict_high`, `world_model_rollout_high`, and `world_model_encode_macro_actions`: individual named method calls and wall time. Calls are nested; their elapsed times overlap and must not be summed. `get_cost` is the outer objective operation. Lower-level neural-network module/kernel counts and FLOPs are not instrumented.
- `environment_step`: a call to installed `stable_worldmodel.World.step()`. Source shows it invokes `get_action` and then one vector-environment `step`. `env_step_index` and monotonic timestamps therefore associate policy observations, solver calls, and environment interactions. It counts calls at the `World` wrapper, not internal physics substeps.
- `environment_transition`: emitted around the exact `world.envs.step(actions)` boundary within `World.step()`. It records the numeric action argument when small enough, the small values and shapes/dtypes of the returned observation/reward/terminated/truncated/info tuple, and the call duration. Large image arrays are represented by shape and dtype. The wrapper calls the original bound method once, returns its exact object unchanged, and restores the vector-env instance method in `finally`.

For flat MPC, observation frequency is the number/timing of policy decisions relative to `World.step`; replanning frequency is solver calls relative to those decisions/steps. The difference in `action_buffer` depth across solver calls gives the executed action chunk. `PlanConfig` gives internal planning horizon and grouped action-block size. For hierarchy, `planner_level` in solver info separates high- and low-level calls, while the resolved config records the high-level replan interval and both plan configurations.

## Smoke check

**Runtime verified:** one call through the real loaded checkpoint -> flat `WorldModelPolicy.get_action` -> CEM -> JEPA `get_cost` -> `encode`/`rollout`/`predict` path completed on CPU with synthetic zero pixel/goal/action tensors. It recorded one policy decision, one CEM solve, one `get_cost` call, two candidate objects scored, one model `encode` for the goal, one for the initial state, one `rollout`, and one `predict`; the returned action had shape `(1, 2)` and the remaining buffer held four actions. There were zero `World.step` calls. The raw details and timing stamps are in the trace file. No success metric was computed.

The smoke `action_block=5` was selected to fit the checkpoint's grouped action encoder. Smoke CEM parameters were `num_samples=2`, `n_steps=1`, `topk=2`, horizon 1 and receding horizon 1. The final wrapper version also completed the same smoke after adding bounded numeric values to small planner tensors and exclusive file creation. These are test settings only and say nothing about research performance. The small latent/image numerical output is not reported as a scientific result.

## Real PushT smoke run (2026-10-01)

**Runtime verified:** one evaluation episode ran to completion through `research/instrumentation/run_traced_hi_eval.py`, the checked-in Hi-LeWM `hi_eval.py`, the real `swm/PushT-v1` environment, the official dataset cache, and the existing base checkpoint. The evaluator returned with exit code 0 after 50 environment steps. Its final policy observation carried `terminated=true`, `truncated=false`; the evaluator produced its one-episode manifest and rollout video. The evaluator labeled this smoke episode `PASS`; this is recorded only as smoke-test behavior and is not a scientific outcome.

The invocation selected the evaluator's existing flat branch and `policy=pusht/lewm`, limited only `eval.num_eval=1`, and used the prepared CPU/headless compatibility settings (`solver.device=cpu`, `MUJOCO_GL=osmesa`, and deletion of unsupported `world.history_size`/`world.frame_skip`). The checked-in flat planner settings remained unchanged: seed 42, CEM population 300, 30 iterations, top-k 30, internal horizon 1, receding horizon 1, and action block 5. The dataset episode selected was episode 1694 at start step 63. These choices establish the execution path only; no comparisons or research interpretation were made.

Trace artifacts:

- Raw event trace: `research/traces/pusht_smoke_20261001T132644Z.jsonl` (1,671 events; 5,466,019 bytes).
- Machine-readable count/timing summary: `research/traces/pusht_smoke_20261001T132644Z_summary.json`.
- Evaluator rollout video and manifest: `/home/bhavana/.stable_worldmodel/pusht/smoke_20261001T132644Z/`.

The trace contains 50 policy calls/observation mappings and 50 `World.step` calls; 10 policy calls replanned. The action buffer refilled to four remaining actions after each solve and drained one per step, confirming a five-action chunk and a solve every five observations. It records 10 CEM solves, each with 30 cost batches and 9,000 candidate objects actually scored; total measured candidate objects scored: 90,000. There were 300 `get_cost`, 300 `rollout`, 300 `predict`, and 600 `encode` calls. No traced call raised an exception. CEM solve duration was 7.41–7.91 seconds (mean 7.76 seconds, CPU); nested world-model timings overlap and include tracing overhead, so they are not additive or benchmark estimates.

The real observations include PushT image tensors of shape `(1, 1, 224, 224, 3)`, proprioception `(1, 1, 4)`, and state `(1, 1, 7)`. Images are fingerprinted rather than embedded in JSONL. That earlier episode's trace did not record numeric action values or post-step returns; the extension below closes those schema gaps, but was validated on a one-call synthetic vector environment rather than by rerunning PushT. Timing fields include wrapper/JSON overhead; nested durations overlap.

**Launcher issue and resolution:** the first launch attempt stopped before data/model/environment setup because importing `hi_eval` as a module made Hydra resolve `./config/eval` as a missing `..config.eval` Python package. No episode or planner call occurred in that attempt. `research/instrumentation/run_traced_hi_eval.py` was minimally changed to compose the same checked-in `hi_pusht` config from its explicit directory, then call `hi_eval.run.__wrapped__` with the same Hydra overrides. No upstream evaluator, YAML config, model, or planner code was modified. The subsequent one-episode run succeeded without further retries.

## Action handoff and transition-return instrumentation

**Runtime verified on a synthetic fixture:** `trace_runtime.py` now temporarily wraps the concrete vector-environment instance's `step` method during each `World.step()` call. At the exact `envs.step(actions)` boundary it records the numeric action values, the returned observation/reward/terminated/truncated/info structure, shapes/dtypes, and elapsed time. It forwards the original action unchanged, invokes the original bound step method once, returns its result unchanged, and restores the original instance method in `finally`. No policy, planner, environment, or model source/behavior was changed.

The one-call fixture used a fake policy and vector environment and wrote `research/traces/action_transition_validation.jsonl`. Assertions confirmed the trace contains action `[[0.25, -0.5]]`, reward `[0.75]`, `terminated=[true]`, and `truncated=[false]`; the fake environment received exactly the same action array. It did not create/step PushT or invoke CEM/model inference. Thus the fields and wrapper forwarding behavior are verified for the fixture; visibility with the actual PushT vector wrapper remains untested until a later authorized run.

## Failed instrumentation attempts retained in the run log

- Passing the existing `lewm_object.ckpt` filename directly to `AutoCostModel` caused its cache-prefix resolver to append `_object.ckpt` again. The smoke launcher now passes the correct prefix; no model or checkpoint was changed.
- The first synthetic action tensor used 2 values per group, but the checkpoint action encoder expects 10 (five 2D actions); CEM failed in the model before returning an action. The smoke input/config was corrected to `action_block=5` and a 10-value grouped action.
- A `topk=1` smoke completed but emitted PyTorch's sample-standard-deviation degrees-of-freedom warning. A clean repeat used `topk=2`; both trace files remain available.

## Remaining unknowns and limitations

- **Still not tested:** the new action/return capture against the actual PushT vector wrapper, hierarchical high/low traces, GPU timings, or instrumentation overhead relative to an uninstrumented run. The real flat PushT observation/policy/CEM/model/step path was exercised before this trace extension. Model methods called during planner assembly/calibration are included by the launcher, but checkpoint deserialization before the policy-build hook is not counted as a model operation.
- **Real PushT smoke preflight (2026-10-01, historical):** the dataset was missing before human authorization. It has since been downloaded and verified under `/home/bhavana/.stable_worldmodel`; see `ENVIRONMENT.md` and the smoke run above. The earlier missing-data check was a preflight blocker, not an evaluator failure.
- **Inference/dataset distinction (source inspected 2026-10-01):** the LeWM weights/config are sufficient to instantiate the JEPA model and call its inference methods when tensors, preprocessing, and a goal are supplied. A `PushT`/`World` can also be constructed and reset independently; PushT creates a goal render on reset and exposes it through wrapped info. The existing `hi_eval.py` and LeWM `eval.py` are not inference-only entrypoints: each calls `HDF5Dataset`, fits action/proprio/state `StandardScaler`s from it, chooses valid episode/start rows, and invokes `World.evaluate_from_dataset`/`World.evaluate` to initialize state and goal from a trajectory. Hi-LeWM's evaluator passes the dataset into policy construction as well; hierarchical mode uses it for latent-prior calibration (and optional empirical macro-action construction). Thus the full official file is required by the checked-in evaluation/replay pipeline and exact preprocessing, but is **not an intrinsic requirement of a model forward pass or PushT environment reset**. A custom one-off inference runner could avoid it only by supplying equivalent preprocessing and an explicit goal/start procedure; this has not been implemented or validated and is not interchangeable with the official evaluator.
- The pretrained checkpoint config contains architecture settings and the weights contain model parameters; the evaluation scripts create preprocessing scalers separately from the dataset. No bundled scaler/statistics artifact was found alongside this checkpoint. The official LeWM setup script maps PushT to the single `quentinll/lewm-pusht` repository, and its official file listing shows `pusht_expert_train.h5.zst`; no smaller official subset/alternate PushT dataset compatible with this exact data path was found. No different dataset will be substituted.
- **WSL storage/cache (updated 2026-10-01):** the WSL-native shared cache is `/home/bhavana/.stable_worldmodel`; the official archive, extracted 46,300,921,856-byte HDF5, and verified base checkpoint are present there. WSL had 949,814,431,744 bytes free after extraction. The installed `stable_worldmodel.data.utils.get_cache_dir()` honors `STABLEWM_HOME`; the HDF5 evaluator path is `<STABLEWM_HOME>/pusht_expert_train.h5`. This single cache is shared by later baseline, Flat RH, hierarchical, and horizon runs.
- **Not measured:** full study inference computation/FLOPs, low-level GPU kernel calls, physical simulator substeps, host CPU utilization, energy, or uninstrumented wall-clock inference. On CUDA, host `perf_counter` durations do not synchronize pending kernels; GPU elapsed time needs CUDA events/profiling and a separately characterized tracing overhead. Current verified environment is CPU-only.
- Observation image values are represented by SHA256 plus dimensions rather than duplicated into JSON. Small numeric state/action fields are included in full. Hashes establish byte identity but do not make image contents human-readable.
- Candidate generation is reported from the verified CEM source loop formula, while candidate scoring is counted from returned cost tensors. For partial/failed solves, generated count is not an observed count.
- `World.step` counts one policy/environment vector-step call. Any internal environment frame skip or substeps need a separate environment-level counter if later configurations use them.
- Hi-LeWM's empirical macro-action solver has its own candidate sampling loop and is not a `CEMSolver`; this hook does not count its generated/evaluated candidate objects. The standard configured high/low planners use CEM. Do not enable empirical macro sampling and claim those counts are covered without adding a separate wrapper.
- The smoke decision's timings include wrapper and JSON logging overhead (nested timings overlap). Treat them only as a tracing functionality check.

## Use for a future evaluator run

The smoke invocation selected the existing flat path, base checkpoint, and one evaluation while preserving the checked-in CEM/plan settings:

```bash
python research/instrumentation/run_traced_hi_eval.py \
  planning.mode=flat policy=pusht/lewm eval.num_eval=1 solver.device=cpu \
  '~world.history_size' '~world.frame_skip' \
  output.subdir=smoke_20261001T132644Z
```

Set `PYTHONPATH` to the pinned baseline submodule and Hi-LeWM checkout, `STABLEWM_HOME=/home/bhavana/.stable_worldmodel`, use the prepared WSL venv and `MUJOCO_GL=osmesa`, and assign a unique `RESEARCH_TRACE_PATH`. This smoke is not a baseline reproduction; no additional episode should be run until separately authorized.

## stable-worldmodel 0.1.0 standalone LeWM tracing (source inspected; not yet run)

The API/checkpoint compatibility candidate is `stable-worldmodel==0.1.0` with `stable-pretraining==0.1.6`. It has `WorldModelPolicy.get_action`, `CEMSolver.solve`, and `World._run_iter`, but no `World.step`. `trace_runtime.py` now retains its 0.0.6 `World.step` path and adds a reversible 0.1.0 `_run_iter` wrapper around the existing vector-env `step` call. The 0.1.0 policy stores per-environment action deques, so the existing aggregate buffer fields now sum those queues and also report per-environment queue lengths.

`research/instrumentation/run_traced_lewm_eval.py` is the external wrapper for the existing standalone `le-wm/eval.py` baseline. It composes the same checked-in `pusht` config, uses `load_pretrained`, captures model/policy/CEM/world boundaries, and leaves upstream evaluator/config code unchanged. `research/remote_preflight.py` and `research/install_remote_cuda_env.sh` support the remote setup. The 0.1.0 API names and evaluator internals were inspected from source, but the new launcher and `_run_iter` trace records have not been executed because no local CUDA device exists. The candidate load test did verify the official state-dict checkpoint strictly; it did not instantiate CEM or run a model forward/episode. See `research/COLAB_MIGRATION.md` for exact staging and the required remote software-only checks.
