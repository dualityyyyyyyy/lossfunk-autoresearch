# Existing PushT Baseline Protocol

This records the repository's current LeWorldModel PushT baseline as checked in. It does not select new study horizons, compute budgets, or alter the frozen research specification. Values below are source/config facts or configuration-derived expectations. An attempt to run the unchanged evaluator on 2026-10-01 stopped at checkpoint-loader lookup before any policy decision or episode; see `research/results/baseline_lewm_reproduction_20261001T1401Z.json`.

## Source configuration and execution

- Baseline evaluator: `le-wm/eval.py`.
- Evaluation config: `le-wm/config/eval/pusht.yaml`.
- Solver config: `le-wm/config/eval/solver/cem.yaml`.
- Repository-documented command: `python eval.py --config-name=pusht.yaml policy=pusht/lewm` from `le-wm/`.
- `policy` defaults to `random` in YAML, so the documented checkpoint override `policy=pusht/lewm` is required.
- Repositories: LeWorldModel `8edfeb336732b5f3ce7b8b210d0ba370a09e2cac`; Hi-LeWM `4bb21a2888e8f22b8d084762c80361e398968775`; pinned LeWorldModel submodule `83f97d72ad067855bc89a1b74b4aff11d4dfdf0c`.

The available pretrained checkpoint is the converted PushT LeWM object checkpoint at `/home/bhavana/.stable_worldmodel/pusht/lewm_object.ckpt`, SHA-256 `91825ebcc183f6119deae85d0c98d82b68b1c0f151769c9d2ba8544b3e2796a0`; its policy/cache name is `pusht/lewm`. The LeWorldModel evaluator source loads through `swm.wm.utils.load_pretrained`, while the Hi-LeWM wrapper loads through `AutoCostModel`. The smoke run validated the Hi-LeWM loader/path, not equivalence of those two loader paths. The repository's standalone LeWorldModel evaluator moves its model to literal `cuda`; this host is CPU-only. These execution compatibility points must be resolved without changing the config before reproduction.

The compatible pinned software environment is Python 3.10.21, `stable-worldmodel==0.0.6`, and `stable-pretraining==0.1.6`. The shared dataset is `/home/bhavana/.stable_worldmodel/pusht_expert_train.h5`, selected by `STABLEWM_HOME=/home/bhavana/.stable_worldmodel`.

### Compatibility follow-up (2026-10-03)

The preceding checkpoint/environment paragraph records the original WSL setup and has these clarifications after the preserved failed baseline attempt. The LeWM standalone evaluator calls `stable_worldmodel.wm.utils.load_pretrained`; the initial `stable-worldmodel==0.0.6` install lacked that API and baseline launch failed before measurement. An isolated `stable-worldmodel==0.1.0` / `stable-pretraining==0.1.6` candidate provides the API and strictly loads the unchanged official `weights.pt`/`config.json` pair. The CPU-only candidate still cannot run the unchanged CUDA evaluator. For that 0.1.0 standalone path, stage the raw official state-dict pair at `$STABLEWM_HOME/checkpoints/pusht/lewm/{weights.pt,config.json}` and the dataset in the cache directory returned by `stable_worldmodel.data.utils.get_cache_dir()`. The converted `pusht/lewm_object.ckpt` is the legacy Hi-LeWM smoke artifact; it is not a substitute for the raw official files in the standalone loader. No baseline result exists. These are loader/cache corrections only; the baseline YAML values above are unchanged.

## Task and environment settings

| Quantity | Existing setting |
|---|---|
| Environment | `swm/PushT-v1` |
| Evaluation dataset | `pusht_expert_train` from the shared HDF5 cache |
| Evaluation count | `eval.num_eval=50`; `world.num_envs=${eval.num_eval}`, so the evaluator uses 50 vectorized environments |
| Per-trajectory action budget | `eval.eval_budget=50` environment steps |
| Environment time limit | `eval.py` sets `world.max_episode_steps = 2 * eval_budget = 100` |
| Dataset goal offset | `eval.goal_offset_steps=25` dataset steps from each sampled start |
| Image size | 224 pixels |
| Initial state/goal | Dataset-selected start/goal rows, applied through `_set_state` and `_set_goal_state` |
| Seed | `seed=42`; used for dataset evaluation-row sampling and wired into the CEM config |

The LeWorldModel evaluator samples dataset starts with its seeded NumPy generator. It calls the policy from `World.step()` once per vector-environment step, providing each active environment's current observation. Thus observation frequency is one policy call per environment step (one batched policy call for the active vector of up to 50 environments).

## Planning and computation

| Quantity | Existing setting and semantics |
|---|---|
| Policy | External `stable_worldmodel.policy.WorldModelPolicy` with CEM and the pretrained LeWM checkpoint |
| PlanConfig horizon | `horizon=5` planner steps |
| Action block | `action_block=5`; each planner step groups/repeats actions over five environment steps |
| Internal planned sequence | `PlanConfig.plan_len = horizon * action_block = 25` environment-action steps |
| Receding horizon | `receding_horizon=5` planner steps |
| Action chunk | `receding_horizon * action_block = 25` environment-action steps |
| Replanning frequency | `WorldModelPolicy` replans when its action buffer empties: nominally every 25 environment steps. A 50-step budget therefore permits two plan chunks/solves per trajectory if it runs to budget; early termination can reduce this. |
| CEM population | `num_samples=300` candidate sequences per environment per iteration |
| CEM iterations | `n_steps=30` |
| Other CEM settings | `topk=30`, `batch_size=1`, `var_scale=1.0`, `device="cuda"` in the repository YAML, `seed=${seed}` |

The configured horizon is five planner steps / 25 action steps, while the evaluation budget is 50 environment steps. Because the current configuration replans after each 25-action plan chunk, it does **not** keep one plan for the entire 50-step task budget. This is the repository's actual baseline config; this document does not silently reinterpret it as full-task open-loop condition A or choose replacement values.

## Counting rules and current observability

The frozen study definition is used: **one planner evaluation is one candidate planning object actually scored**. The pinned CEM source returns a cost tensor with one entry per candidate and environment. Count `costs.numel()` from each successful `get_cost` call; do not count a `get_cost` invocation as one candidate.

For one environment, the configured CEM scores `300 * 30 = 9,000` candidate objects per solve. With 50 environments and `batch_size=1`, the solver processes 50 environment batches per iteration: 1,500 `get_cost` calls and 450,000 candidate-object scores per vectorized solve. Two full-budget plan chunks would therefore yield 900,000 candidate-object scores across the 50 trajectories. These are configuration/source-derived counts, not measured baseline results; the actual run count can be lower if episodes terminate early.

The runtime trace currently records individual counts/shapes/timings for `get_cost`, `encode`, `predict`, `rollout`, and available high-level model methods, along with CEM solves, cost batches, scored candidate counts, policy observations, action-buffer/replan behavior, environment actions, and post-step returns. Model-operation timings are nested and overlap. The scalar `N_WM` accounting definition remains **unfinalized**; no FLOP count, neural-network kernel count, or equivalent single compute total is currently available. Model-loading/deserialization before the tracing hook is not counted as an inference operation.

## Configuration distinction and evidence status

The real smoke used Hi-LeWM's `hi_pusht.yaml` flat branch with `horizon=1`, `receding_horizon=1`, `action_block=5`, one evaluator episode, and CPU compatibility. That smoke configuration is **not** the LeWorldModel baseline config recorded above. In particular, its five-action chunk and smoke replanning cadence must not be copied into the baseline protocol. No task success result from the smoke is used here.

The baseline values above are verified from checked-in YAML/source and the pinned `stable-worldmodel` implementation. The action-buffer/chunk mechanics were also exercised in the one-episode Hi-LeWM flat smoke, but the LeWorldModel `eval.py` baseline command/config has not been reproduced. The frozen study asks for full-task open-loop A, flat receding-horizon B, and hierarchical C; the present repository baseline settings alone do not establish all three conditions or select their future matched compute budgets.
