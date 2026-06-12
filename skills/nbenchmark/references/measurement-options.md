# MeasurementOptions Reference

`MeasurementOptions` (a `record` with `init` properties) controls every measurement pass. Property setters validate their inputs and throw `ArgumentOutOfRangeException` for out-of-range values. The analyzers package also flags out-of-range literals at compile time (NB0009).

```csharp
var options = new MeasurementOptions
{
    Iterations = 500,
    WarmupIterations = 50,
    MeasureAllocations = true,
    OutlierMode = OutlierMode.IqrFence,
    ConfidenceLevel = 0.99,
    SignificanceLevel = 0.01,
};
```

Constants: `MeasurementOptions.MinIterations` (0), `MeasurementOptions.MaxIterations` (100,000), `MeasurementOptions.MaxWarmupIterations` (10,000). `MeasurementOptions.Default` is a ready-made default instance.

## How each mode sets options

| Mode | How |
|---|---|
| Quick | Pass a `MeasurementOptions` to `Benchmark.Run(..., options: ...)` |
| Suite | Fluent `With*` methods (each updates one option via a `with` expression) |
| Host | `WithOptions(new MeasurementOptions { ... })`; CLI flags override specific fields |

## Options

### Iterations — default `200`, range `0`–`100,000`

Number of measured iterations. More iterations → tighter confidence interval (smaller Error) at the cost of run time. Increase to 500–1000 when the Error column is large relative to the mean, or for sub-microsecond operations. CLI: `--iterations <n>`.

### WarmupIterations — default `25`, range `0`–`10,000`

Iterations run (and discarded) before measurement so the JIT compiles the body and CPU caches warm up. Set to `0`/`1` to measure cold-start behaviour. Increase (e.g. 50) when laptops thermal-throttle or there are many generic instantiations to JIT. CLI: `--warmup <n>`.

### ForceGcBeforeEachIteration — default `true`

Triggers a gen-0 GC collection before each warmup and measured iteration so allocations from a previous iteration don't perturb the next. Disable only when the benchmark intentionally exercises GC/allocation-heavy paths where cumulative effect is the point.

### MeasureAllocations — default `false`

Samples `GC.GetAllocatedBytesForCurrentThread` around each iteration and reports mean bytes/op (Alloc/op column), with a process-wide fallback (`GC.GetTotalAllocatedBytes`) for async thread hops. Adds a small overhead. Suite fluent method: `.WithAllocations()`.

### OutlierMode — default `OutlierMode.IqrFence`

Which samples are discarded before statistics are computed. Trimming happens after timing, before stats; the removed count is recorded on `OutliersRemoved`.

| Value | Behaviour | When to use |
|---|---|---|
| `None` | Keep all samples (sort only) | Latency-tail analysis where every sample matters |
| `RemoveTop5Percent` | Trim slowest 5% | A fixed quota; always drops the slowest 5% |
| `RemoveTopAndBottom5Percent` | Trim 5% from each end | Fast outliers (e.g. cache hits) also skew results |
| `IqrFence` | Remove samples outside `Q1 − 1.5×IQR` … `Q3 + 1.5×IQR` **(default)** | General purpose; adapts to each run's spread |

`IqrFence` keeps nearly every sample on a clean run and trims more on a noisy run. When discarded slow samples form a tight secondary cluster (a possible second execution profile rather than scattered noise), a non-fatal **bimodal-distribution warning** is added to `BenchmarkResult.Warnings`. `LowerFence` / `UpperFence` are populated only in this mode. Suite fluent method: `.WithOutlierMode(mode)`.

### ConfidenceLevel — default `0.95`, range `>0` and `<1`

Confidence level for the margin of error on the mean (the Error column). `0.90` is narrower/less conservative; `0.99` is wider/more conservative. A higher level produces a larger Error. Suite: `.WithConfidenceLevel(0.99)`; CLI: `--confidence 0.99`.

The margin is `StudentT.CriticalValue(confidenceLevel, n−1) × StandardError`, where `StandardError = StandardDeviation / √n`. The true mean lies within `Mean ± MarginOfError` at the given level.

### EnableSignificance — default `true`

When `true` and there are ≥2 benchmarks, runs a two-sided Mann-Whitney U test against the baseline. Disable with `.WithSignificance(false)` to skip the overhead.

### SignificanceLevel — default `0.05`, range `>0` and `<1`

The alpha threshold a p-value must fall below to be reported as **Significant**. Lower it (e.g. `0.01`) to demand stronger evidence before calling a difference real. This is independent of `ConfidenceLevel` (which only affects the Error column). Suite: `.WithSignificanceLevel(0.01)`; CLI: `--alpha 0.01`.

### ForceGcBetweenBenchmarks — default `true`

Runs a full gen-2 GC collection between benchmarks in a suite to prevent carry-over heap state. Keep on unless you have a specific reason to disable.

## Per-method overrides (Host mode)

In Host mode, `[Benchmark(Iterations = 1000, WarmupIterations = 100)]` overrides the host-level options for that one method. Internally these use `-1` as the "unset" sentinel (range otherwise 0–100,000 / 0–10,000). See the `nbenchmark-host` skill.

## Dry run

`Iterations = 0` **and** `WarmupIterations = 0` together are the dry-run signal: the body is not invoked and a zeroed result is returned. CLI `--dry-run` is exactly `--iterations 0 --warmup 0`. To invoke the body once as a smoke test, use `--iterations 1 --warmup 0` instead.
