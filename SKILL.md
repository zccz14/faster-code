---
name: faster-code
description: Use this skill whenever code is slow, times out, gets killed, burns substantial CPU, runs long batch jobs, performs parameter scans, trains models, processes large datasets, runs simulations, or has unknown target-scale runtime. It guides progressive runtime probing, fixed-timeout sampling, N-T scaling estimation, and full-run go/no-go decisions before attempting target scale. If optimization happens, it requires preserving slow-version canonical output and proving the faster version is byte-for-byte identical after pre-write normalization such as rounding or formatting. This skill does not wrap profiler tools and does not provide generic optimization recipes; it decides whether to run full scale, stop, or move into profiling and optimization.
---

# faster-code

Use this skill to improve how compute-heavy code is run: first decide whether the target scale can finish within the runtime budget, then decide whether to run full scale, stop, or optimize. Do not start by throwing a slow program at a long timeout. A long timeout only proves the waste already happened; short bounded probes produce decision information early.

## Core Goal

Turn “can this finish at full scale?” into a pre-run gate:

1. Define the target scale `N_full` and acceptable runtime budget `T_budget`.
2. Build 3-5 valid `N-T` samples using short timeouts, progress logs, or sliced inputs.
3. Estimate target-scale runtime with multiple simple models.
4. If the estimate is unacceptable, do not run full scale; move to profiling, diagnosis, or optimization.

If code is optimized, add a semantic gate: preserve canonical output from the slow version as the ground truth, then require the faster version to produce byte-for-byte identical canonical output on the same input slice. If floating-point differences are natural, normalize values before writing canonical output using a predeclared rounding or formatting rule. It is acceptable to lose limited precision for auditability, but the final comparison must still be byte-for-byte.

## Classify the Program

### A. Progress-Observable Programs

If the program can print progress while running, prefer one short timeout probe instead of many sliced runs.

Useful progress logs are stable, low-frequency, and parseable:

```text
PERF_PROGRESS processed=120000 total=1000000 elapsed_sec=31.2 rate=3846.1
```

Procedure:

1. Run with a short timeout, usually 30 seconds.
2. Extract multiple `elapsed_sec -> processed_N` points from logs.
3. Treat a timeout-killed run as useful if progress logs were captured.
4. If throughput declines over time, extrapolate conservatively; do not extrapolate from the fastest early rate.

If modifying the program is cheap, add progress logging before doing repeated sliced runs. Log every 5-10 seconds, not for every item.

### B. Batch-Only Programs

If the program only reports completion after the whole algorithm finishes, use sliced data and repeated bounded runs.

Procedure:

1. Identify the control for input scale, such as `--limit`, date range, file count, sample ratio, batch count, or parameter count.
2. Start from a small `N` and expand exponentially, for example 1k, 2k, 4k, 8k.
3. When a run times out, binary-search between the previous completed `N` and the failed `N`.
4. Use binary search to construct reliable `N-T` samples, not to find the exact largest possible `N`.
5. Do not waste runs trying to hit an exact second value; the goal is a scaling trend, not a precise benchmark.

## Define N

Before extrapolating, define the main scale variable `N`. Do not assume `N` is always row count.

Common definitions include:

- input rows;
- files or objects processed;
- events or tasks generated;
- parameter combinations;
- workers, shards, or independent units;
- training windows or batches;
- a combined variable such as `rows x parameter_combinations x files`.

If the program has multiple dominant scale variables, use the combined variable that best explains the work. If `N` cannot represent the full-scale computation, mark the result `INCONCLUSIVE`.

## Sampling Requirements

Default rules:

- Require at least 3 completed sample points before extrapolating to full scale.
- Prefer 3-5 completed sample points.
- More than 5 points is usually unnecessary unless the samples are unstable.
- Start with a 30 second timeout for probes.
- Avoid probe nodes longer than about 180 seconds unless the user explicitly accepts the cost.

Useful time bands:

- 30 second node: completed samples in 20-40 seconds are acceptable.
- 60 second node: completed samples in 45-75 seconds are acceptable.
- 120 second node: completed samples in 90-150 seconds are acceptable.
- 180 second node: completed samples in 140-220 seconds are acceptable.

Timeout samples mean `T(N) > timeout`. Do not fit them as if `T(N) = timeout`.

Record at least:

```text
N:
runtime_sec:
status: completed | timeout | failed
timeout_sec:
command:
slice_method:
notes:
```

Also record peak memory, CPU utilization, output size, and cache state when available.

## Extrapolation Methods

Do not use a single estimate. Compare at least these three views and make a conservative decision.

### 1. Linear Extrapolation

```text
T_full = T_last * N_full / N_last
```

This is a lower-bound check. If even the linear estimate exceeds the budget, usually stop.

### 2. Power-Law Fit

```text
T = a * N^p
log(T) = log(a) + p * log(N)
```

Use it to estimate the scaling exponent `p`.

Default interpretation:

- `p <= 1.2`: approximately linear.
- `1.2 < p <= 1.6`: cautious; require the target estimate to be comfortably below budget.
- `1.6 < p <= 2.2`: high risk; usually optimize or reduce scope first.
- `p > 2.2`: do not blindly run full scale unless `N_full` is very small.

### 3. Local-Slope Extrapolation

Use only the largest two or three completed `N` samples to estimate late-stage growth. This catches cases where small samples are fast but larger samples degrade.

If linear, power-law, and local-slope estimates disagree strongly, mark the result `INCONCLUSIVE` or `FAIL`. Do not approve full scale using the most optimistic estimate.

## Go/No-Go Decision

Return exactly one of:

```text
PASS: Target scale may be run.
FAIL: Do not run target scale; diagnose or optimize first.
INCONCLUSIVE: Evidence is insufficient or unstable; do not run target scale yet.
```

Default failure conditions:

- Fewer than 3 completed sample points.
- `N` does not represent the dominant computation.
- Linear extrapolation already exceeds `T_budget`.
- Conservative estimate exceeds `T_budget` by more than 1.5x.
- Scaling exponent is clearly worse than linear and `N_full` is far beyond the sampled range.
- Samples are non-monotonic or unstable without explanation.
- Probes suggest memory, output size, or I/O may become the bottleneck.

Default pass conditions:

- Samples are stable.
- `N` is credible.
- Multiple estimates are below `T_budget`.
- The largest sample is not too far from `N_full`, or the scaling is close to linear.

## Semantic Gate After Optimization

When the task moves from “can it finish?” to “make it faster,” protect the slow version’s semantics first. The slow version may be slow, but it is often clearer and more trustworthy. A faster version without a semantic check may simply produce different results faster.

Procedure:

1. Before changing code, choose a small or medium sample that completes and covers key paths.
2. Generate canonical output with the slow version and save it where the faster version cannot overwrite it.
3. Define canonical fields, sort order, floating-point format, timestamp format, null representation, and randomness controls.
4. If floats are present, define fixed rounding or formatting before writing canonical output, including decimal places, scientific notation, NaN/Inf representation, and negative zero handling.
5. Generate candidate canonical output with the faster version using the same input, same parameters, and same environment assumptions.
6. The final comparison must be byte-for-byte identical.
7. Do not replace canonical normalization with runtime tolerance comparisons. Tolerance belongs only in the pre-write rounding or formatting rule.
8. Until the semantic check passes, the faster version must not replace the slow version or support full-scale conclusions.

Canonical output should be small and stable. Include only what proves semantic equivalence, such as:

- final metrics and key intermediate metrics;
- per-item, per-task, or per-result core outputs;
- sorted IDs, timestamps, scores, states, labels, or decisions;
- required aggregate checksums or summary rows.

Do not use huge temporary files as canonical output. Canonical output is for auditing semantics, not copying every artifact.

Additional pass conditions after optimization:

- Slow-version canonical output exists and is preserved.
- Faster-version candidate canonical output exists.
- Byte-for-byte comparison passes.
- Runtime sampling shows the faster version improves target-scale feasibility without new unexplained instability.

Additional failure conditions after optimization:

- No slow-version canonical output exists.
- Faster-version canonical output is not byte-for-byte identical.
- Floats were not normalized before writing canonical output.
- Rounding or formatting rules were changed after seeing differences.
- Only aggregate metrics are similar while important per-item outputs drift.

## Profiler Guidance

This skill does not wrap or prescribe profiler tools. However, when the gate returns `FAIL` or `INCONCLUSIVE`, especially for high complexity, declining throughput, suspected memory bottlenecks, or unstable samples, tell the user to use the existing profiler or hotspot audit tool appropriate for the project and language.

Profiler use is a next-stage input, not a replacement for this gate. Keep profiling short, controlled, and reproducible:

- Profile a small or medium sample that reliably hits the slow path.
- Set an explicit timeout so profiling does not become another blind full-scale run.
- Capture hot functions, hot lines, call counts, cumulative time, allocation pressure, or I/O waits.
- Do not rewrite semantic-sensitive code from profiler results unless canonical output exists.
- After optimization, return to this skill: pass the canonical byte-for-byte gate, then repeat `N-T` sampling and the full-scale go/no-go decision.

## Report Format

Use this structure:

```text
Performance Gate Result: PASS | FAIL | INCONCLUSIVE

Target:
- N_full:
- T_budget:
- command:
- N_definition:

Program Type:
- progress_observable | batch_only
- sampling_method:

Samples:
| N | runtime_sec | status | timeout_sec | notes |

Scaling:
- linear_estimate:
- power_law_p:
- power_law_estimate:
- local_slope_estimate:
- conservative_estimate:

Canonical Check, if optimized code was created:
- slow_canonical:
- fast_candidate_canonical:
- compare_rule:
- compare_result:
- semantic_gate:

Decision:
- conclusion:
- reason:
- next_step:
```

## Non-Goals

This skill does not:

- wrap profiler tools;
- provide generic optimization recipes;
- automatically rewrite algorithms;
- prove estimates are exact;
- encourage full-scale runs when evidence is insufficient.

If the gate fails, recommend profiling, diagnosis, algorithm changes, progress logging, a better slicing strategy, or redefining `N`. Do not start a long target-scale run.

## Operating Rule

When the user asks to run full scale, check whether something will time out, or make slow code faster, use this skill before long execution. Treat timeouts as sampling signals, not final answers. The goal is to get decision information early and avoid wasting compute on runs that were predictable failures.
