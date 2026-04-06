---
name: metaopt-results-analysis
description: "Use when the ml-metaoptimization orchestrator receives completed batch results. Compares results against the baseline, judges improvement or regression, extracts key learnings, and identifies proposal invalidations. Keywords: results analysis, experiment evaluation, baseline comparison, learnings extraction, metaoptimization worker."
---

# metaopt-results-analysis

## Overview

Compare completed batch results against the aggregate baseline and extract learnings. This is the evaluation step that determines whether an experiment improved the campaign metric and what knowledge was gained.

This skill is a leaf worker in the `analysis` auxiliary slot. It is dispatched by the `ml-metaoptimization` orchestrator after the backend reports a batch as `completed` and the orchestrator fetches results via `results_command`. It does not schedule itself, access the queue, or modify campaign state — it receives inputs, performs analysis, and returns structured output.

**Lane:** Auxiliary slot — `analysis`
**Model class:** `strong_reasoner` (prefer a strong reasoning model like Opus 4.6 fast, fallback to a capable general model like GPT-5.4)

## Input Contract

The orchestrator provides all inputs via the subagent prompt. Do not assume access to any files, state, or campaign configuration beyond what is passed in.

### Standard Envelope (provided by orchestrator on every dispatch)

| Field | Type | Description |
|-------|------|-------------|
| `campaign_id` | string | Campaign identifier |
| `current_iteration` | integer | Current iteration number |
| `slot_id` | string | The slot ID dispatching this worker |
| `attempt` | integer | Attempt number for this dispatch (1-indexed) |

### Batch Results Payload

The results payload comes from the backend's `results_command` and follows the results contract shape:

- `batch_id` — the batch identifier echoed from the manifest
- `status` — must be `"completed"`
- `best_aggregate_result.metric` — the metric name (must match the campaign objective)
- `best_aggregate_result.value` — the numeric aggregate score
- `per_dataset` — per-dataset breakdown of results
- `artifact_locations` — pointers to code, data manifest, and logs
- `logs_location` — execution log pointer

### Campaign Objective

- `metric` — the metric being optimized
- `direction` — `minimize` or `maximize`
- `aggregation_method` — aggregation method (e.g., `weighted_mean`, `mean`)
- `aggregation_weights` — per-dataset weights (when method is `weighted_mean`; `null` otherwise)
- `improvement_threshold` — minimum magnitude of change to qualify as improvement

### Baselines

- `aggregate` — the current aggregate baseline value
- `by_dataset` — per-dataset baseline values

### Experiment Context

- The experiment design that was tested (from `selected_experiment`)
- The winning proposal that motivated the experiment
- Key learnings from prior iterations (`key_learnings`)
- Completed experiments history (`completed_experiments`)

## Output Contract

Return a structured response containing all of the following sections.

### Judgment

Exactly one of:

| Judgment | Condition |
|---|---|
| `improvement` | Aggregate metric improved beyond `improvement_threshold` in the objective `direction` |
| `regression` | Aggregate metric worsened in the objective `direction` |
| `neutral` | Change magnitude is within `improvement_threshold` |

### Updated Aggregate Baseline

- If judgment is `improvement`: return the new aggregate value as the updated baseline
- If judgment is `regression` or `neutral`: return the existing baseline unchanged

### Per-Dataset Analysis

For each dataset in `per_dataset`, report:

- Dataset identifier
- Previous baseline value
- New result value
- Delta (signed, in the objective direction)
- Individual judgment: `improved`, `regressed`, or `flat`

### Updated Learnings

New key insights to carry forward, appended to the existing `key_learnings`. Each learning must be:

- **Concrete:** references specific parameters, techniques, or configurations
- **Actionable:** implies what to try or avoid next
- **Evidenced:** cites the result data that supports the conclusion

Learnings must cover:

- What worked and why (for improvements)
- What didn't work and why (for regressions or neutral results)
- Implications for future proposals
- Dataset-specific observations when per-dataset results diverge from the aggregate

### Proposal Invalidations

Identify proposals in the upcoming pool that should be deprioritized or removed based on these results. Each invalidation must:

- Name the specific proposal or proposal pattern
- Cite evidence from the results (e.g., "technique X was the core of this experiment and produced regression on 3/4 datasets")
- Recommend `deprioritize` or `remove`

If no proposals should be invalidated, return an empty list and state why.

### Carry-Over Candidates

Aspects of the current experiment that deserve further exploration in future proposals. Examples:

- A technique that improved some datasets but not others — worth testing in isolation
- A hyperparameter range that showed a trend — worth narrowing
- A combination that was neutral overall but showed promise on key datasets

If nothing warrants carry-over, return an empty list and state why.

### Persistence Note

The orchestrator persists the structured output as `state.selected_experiment.analysis_summary` with the following shape:
- `judgment`: the improvement/regression/neutral verdict
- `new_aggregate`: the post-experiment aggregate score
- `delta`: signed change from baseline
- `learnings`: array of new key insights (appended to `state.key_learnings`)
- `invalidations`: proposal invalidation recommendations (forwarded to rollover)
- `carry_over_candidates`: aspects worth further exploration (forwarded to rollover)

Structure your output to match this shape so the orchestrator can persist it without transformation.

## Judgment Criteria

### Computing the Aggregate Score

1. Use the campaign's declared `aggregation_method`
2. For `weighted_mean`: compute `sum(weight_i * value_i) / sum(weight_i)` using `aggregation_weights`
3. The metric name in the results must match `objective.metric` — if it doesn't, report an error rather than guessing

### Applying the Threshold

1. Compute `delta = new_aggregate - baseline_aggregate`
2. For `direction = maximize`: improvement when `delta > improvement_threshold`
3. For `direction = minimize`: improvement when `delta < -improvement_threshold` (i.e., the value decreased beyond the threshold)
4. Regression is the opposite direction beyond zero
5. Neutral is `|delta| <= improvement_threshold`

### Direction Consistency

All comparisons must respect the objective direction. "Better" means higher when maximizing, lower when minimizing. Never invert this.

## Behavioral Rules

1. **Use `aggregation_method` and `aggregation_weights`** to compute the aggregate score. Do not substitute a different aggregation.
2. **Compare against the declared improvement_threshold.** Small numerical changes below threshold are `neutral`, not `improvement`.
3. **Respect the objective direction** (`minimize` vs `maximize`) in all comparisons and language.
4. **Do not cherry-pick per-dataset results** to claim improvement when the aggregate regressed. The aggregate judgment is authoritative.
5. **Learnings must be concrete and actionable**, not generic platitudes like "more experiments needed" or "results are promising."
6. **Proposal invalidations must cite specific evidence** from the results. Do not invalidate proposals speculatively.
7. **Metric name mismatch is an error.** If `best_aggregate_result.metric` does not match `objective.metric`, report the mismatch instead of proceeding with analysis.
8. **Do not modify or reinterpret the results payload.** Use the values as provided by the backend.
9. **Per-dataset analysis is supplementary to the aggregate judgment.** It provides detail but does not override the aggregate.
10. **When per-dataset results diverge** (some improve, some regress), explain the divergence in learnings rather than averaging away the nuance.

## Common Mistakes

- **Inverting the direction:** Treating a decrease as improvement when `direction = maximize`. Always check.
- **Ignoring the threshold:** Declaring improvement for a delta of 0.001 when threshold is 0.01.
- **Vague learnings:** "The model performed slightly better" — instead state "Learning rate 3e-4 with cosine schedule improved val_loss by 0.02 on dataset-A but regressed dataset-B by 0.01, suggesting the schedule benefits smaller datasets."
- **Cherry-picking:** "Dataset-A improved by 5%!" when the aggregate regressed. The aggregate judgment must come first.
- **Unsupported invalidations:** "We should remove all learning-rate proposals" without evidence from this experiment's results.
- **Fabricating analysis:** If the results payload is incomplete or malformed, report the issue rather than inventing plausible numbers.
- **Forgetting carry-over on neutral results:** Neutral doesn't mean nothing was learned. Partial per-dataset signals often suggest refinements.

## References

- `ml-metaoptimization/references/worker-lanes.md` — authoritative analysis lane contract (inputs, outputs, purpose)
- `ml-metaoptimization/references/backend-contract.md` — results payload shape (`best_aggregate_result`, `per_dataset`, `artifact_locations`)
- `ml-metaoptimization/references/contracts.md` — state field definitions (`baseline`, `key_learnings`, `completed_experiments`, `objective_snapshot`)
