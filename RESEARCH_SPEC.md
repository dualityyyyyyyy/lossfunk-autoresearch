# Frozen Research Specification

## Research question

"When controlling for local replanning and inference computation, does explicit hierarchical subgoal representation provide an additional reliability advantage over flat receding-horizon world-model planning, and how does this advantage scale with task horizon?"

## Hypothesis

Explicit hierarchical subgoal representations may provide an additional reliability advantage at sufficiently long task horizons, but not necessarily at short horizons.

## Primary environment

PushT.

## Required comparison

A. **Flat open-loop planning:** planning over the complete task horizon without intermediate replanning.

B. **Flat receding-horizon planning:** planning over a short horizon, executing part of the plan, observing the updated state, and replanning.

C. **Hierarchical planning:** a high-level planner predicts intermediate subgoals and a low-level planner reaches them.

## Critical comparison

Hierarchical planning versus flat receding-horizon planning.

The comparison must control or explicitly measure:

- total inference computation
- world-model forward passes
- planner evaluations
- planning iterations
- model parameter count
- wall-clock inference time
- environment interactions
- planning horizon
- replanning frequency
- execution block size
- random seed

## Primary analysis

Define `Delta(H,C) = S_hier(H,C) - S_flat-RH(H,C)`.

Test whether the incremental hierarchical advantage changes with task horizon `H` at approximately matched inference-computation budget `C`.

## Primary metrics

- task success rate versus planning horizon
- task success rate versus inference computation
- subgoal-reaching success
- final task-state error
- number of model evaluations
- inference time

## Diagnostics

- one-step prediction error
- multi-step prediction error
- subgoal prediction error
- subgoal reachability
- subgoal completion
- cumulative reward

## Failure categories

- incorrect world-model prediction
- poor subgoal selection
- poor subgoal reachability
- low-level controller failure
- planning-search failure
- compounding error

## Minimum successful study

1. Establish/reproduce a flat world-model planning baseline on PushT.
2. Implement/validate flat receding-horizon planning.
3. Implement/validate hierarchical planning with learned subgoals.
4. Evaluate across multiple short, medium, and long horizons.
5. Perform the compute-matched comparison.
6. Use multiple random seeds where computationally feasible.
7. Preserve all raw code, logs, configurations, results and figures.

## Secondary experiments only

- oracle subgoals
- noisy subgoals
- subgoal reachability analysis
- second environment

Do not let secondary experiments delay the minimum successful study.

## Scientific integrity rules

- Do not invent or estimate experimental results and present them as measurements.
- Do not silently change the research question.
- Do not change the primary comparison to make the experiment easier.
- Do not call a comparison compute-matched unless the relevant computation has actually been measured.
- Do not treat oracle subgoals as learned subgoals.
- Do not delete failed experiments or failed runs.
- Preserve raw outputs.
- Clearly distinguish implementation failures from scientific negative results.
- If the full experiment is infeasible, reduce scope while preserving the core hierarchical-versus-flat-receding-horizon comparison and document the reduction.

The autoresearch artifact must preserve failures rather than polishing them away.
