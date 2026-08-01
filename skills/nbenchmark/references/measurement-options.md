# MeasurementOptions Reference

`MeasurementOptions` (a `record` with `init` properties) controls every measurement pass. Property setters validate their inputs and throw `ArgumentOutOfRangeException` for out-of-range values. The analyzers package also flags out-of-range literals at compile time (NB0009).

```csharp
var options = new MeasurementOptions
{
    Iterations = 500,
    WarmupIterations = 50,
    ConfidenceLevel = 0.99,
    SignificanceLevel = 0.01,
};
```

`Iterations`, `WarmupIterations`, and `OpsPerSample` are `int?`: `null` (the default) means auto-resolve. Constants: `MeasurementOptions.MinIterations` (0), `MeasurementOptions.MaxIterations` (100,000), `MeasurementOptions.MaxWarmupIterations` (10,000), `MeasurementOptions.MaxOpsPerSampleLimit` (16,777,216), `MeasurementOptions.MinHistogramBucketCount` (5), `MeasurementOptions.MaxHistogramBucketCount` (100), `MeasurementOptions.DefaultMinimumPracticalEffect` (0.147), `MeasurementOptions.DefaultMaxRawSamples` (4096), `MeasurementOptions.UnboundedRawSamples` (0). The replicate-count ceiling lives on the `LaunchCounts` static class (`LaunchCounts.Max = 100`), not on `MeasurementOptions` - see [LaunchCount](#launchcount) below. `MeasurementOptions.Default` is a ready-made default instance (all three counts `null`). `MeasurementOptions.For(profile)` is a factory that sets `Profile`.

## How each mode sets options

| Mode | How |
|---|---|
| Single | Pass a `MeasurementOptions` to `Benchmark.Run(..., options: ...)` |
| Suite | Fluent `With*` methods (each updates one option via a `with` expression) |
| Harness | `WithOptions(new MeasurementOptions { ... })`; CLI flags override specific fields |

## Options

### Iterations — `int?`, default `null` (auto), range `0`-`100,000`

Measured-sample count. `null` (the default) lets the adaptive loop stream samples until the confidence interval on the mean meets its relative-width target. `0` is a dry-run (see below). A positive value pins an exact sample count — use it for fully reproducible runs, or raise it for sub-microsecond operations when you want a fixed budget. CLI: `--iterations <n>`.

### WarmupIterations — `int?`, default `null` (auto), range `0`-`10,000`

Warmup samples run (and discarded) before measurement so the JIT compiles the body and CPU caches warm up. `null` (the default) lets a plateau detector end warmup once timings stop improving. `0` skips warmup to measure cold-start behaviour. A positive value pins an exact warmup count. CLI: `--warmup <n>`.

### OpsPerSample — `int?`, default `null` (auto), range `1`-`16,777,216`

How many times the body is invoked per timed sample (**K**). `null` (the default) auto-calibrates K for fast, side-effect-free bodies so a single timer read spans roughly `AutoTune.TargetSampleDurationNs` of work; the reported per-op time and allocations are divided back down by K. A positive value pins K. Calibration is skipped when an iteration setup/teardown is present (the body isn't safely repeatable), leaving K at 1. CLI: `--ops-per-sample <n>`.

### AutoTune — default `AutoTuneOptions.Default`

Bounds and steers the adaptive loop: `MinWarmup`/`MaxWarmup`, `WarmupEpsilon`, `PlateauPatience`, `MinWarmupTime`, `RequireJitQuiescence`, `MinSamples`/`MaxSamples`, `CiTarget` (relative CI half-width target, default `0.025`), `TargetSampleDurationNs`, `MaxOpsPerSample`, `BatchSize`, `MaxTuningTime` (overall wall-clock cap), `EnableJitterCalibration`, `JitterCalibrationSamples`, `JitterCalibrationWorkPerSample`, `JitterAutoSwitchThreshold`, `CapBehavior` (`Warn` / `Error`), `WarmupBudgetFraction`, `CapGraceFactor`. Use a preset — `AutoTuneOptions.Quick`, `AutoTuneOptions.Default`, or `AutoTuneOptions.Thorough` — or build your own (`AutoTuneOptions.FromPreset(preset)`). Suite/Harness: `.WithAutoTune(preset)` or `.WithAutoTune(options)`; CLI: `--auto-tune <default|quick|thorough>` plus `--ci-target`, `--min-samples`, `--max-samples`, `--min-warmup`, `--max-warmup`, `--max-tuning-time`.

### Diagnostics — default `DiagnosticsOptions.Default` (GC collection counts on)

Runtime counters collected during measurement. `DiagnosticsOptions` is an init-only record with `GcCollectionCounts` (default true), `GcHeapInfo` (default false), `Exceptions` (default false), `CpuTime` (default false). Presets: `DiagnosticsOptions.Default`, `DiagnosticsOptions.All`, `DiagnosticsOptions.None`. Convert to/from a flags enum with `ToMode()` / `FromMode(mode)`. Suite/Harness: `.WithDiagnostics(options)` or `.WithDiagnostics(mode)`; the `DiagnosticsMode` flags enum has named presets `Gc`, `GcAndCpu`, `All`.

### Profile — default `MeasurementProfile.Realistic`

The authoritative measurement profile. The resolved GC booleans (`ForceGcBeforeEachIteration`, `ForceGcBeforeMeasurement`, `ForceGcBetweenBenchmarks`, `MeasureAllocations`) derive from this unless an explicit override is set. `Realistic` (the default) matches production - no forced GC between iterations, warmup heap carries into measurement. `Independent` forces a Gen0 GC before each iteration and a full GC between warmup and measurement so each sample sees a clean heap. CLI: `--profile <realistic|independent>`.

### ForceGcBeforeEachIteration — computed from `Profile`

Whether a Gen0 GC is forced before each warmup and measured iteration. `true` under `MeasurementProfile.Independent`, `false` under `Realistic` (the default). Override with `ForceGcBeforeEachIterationOverride` (a `bool?`); CLI: `--force-gc` forces on. Disable only when the benchmark intentionally exercises GC/allocation-heavy paths where cumulative effect is the point.

### ForceGcBeforeMeasurement — computed from `Profile`

Whether a full GC runs once between warmup and measurement, clearing the warmup heap so it cannot trigger a collection mid-measurement. Forced under `Independent`, off under `Realistic`. Override with `ForceGcBeforeMeasurementOverride`.

### ForceGcBetweenBenchmarks — default `true`

Whether a full GC runs between benchmarks so one benchmark's leftover heap cannot bias the next (which would make results order-dependent and undermine the significance test's independence assumption). Override with `ForceGcBetweenBenchmarksOverride`.

### MeasureAllocations — default `true`

Whether per-iteration allocations are sampled and reported (Alloc/op column), with a process-wide fallback (`GC.GetTotalAllocatedBytes`) for async thread hops. On by default under both profiles; override with `MeasureAllocationsOverride = false` to disable. Suite fluent method: `.WithAllocations(bool = true)`; CLI: `--no-allocations` disables.

### OutlierMode — default `OutlierMode.IqrFence`

Which samples are discarded before statistics are computed. Trimming happens after timing, before stats; the removed count is recorded on `OutliersRemoved`.

| Value | Behaviour | When to use |
|---|---|---|
| `None` | Keep all samples (sort only) | Latency-tail analysis where every sample matters |
| `RemoveTop5Percent` | Trim slowest 5% | A fixed quota; always drops the slowest 5% |
| `RemoveTopAndBottom5Percent` | Trim 5% from each end | Fast outliers (e.g. cache hits) also skew results |
| `IqrFence` | Remove samples outside `Q1 − 1.5×IQR` ... `Q3 + 1.5×IQR` **(default)** | General purpose; adapts to each run's spread |
| `MedianAbsoluteDeviation` | Remove samples outside `median ± threshold × MAD` (scaled by 1.4826; default `threshold = 3.0`) | Robust to extreme outliers |

`IqrFence` keeps nearly every sample on a clean run and trims more on a noisy run. When discarded slow samples form a tight secondary cluster (a possible second execution profile rather than scattered noise), a non-fatal **bimodal-distribution warning** is added to `BenchmarkResult.Warnings`. `LowerFence` / `UpperFence` are populated only in `IqrFence` mode. Suite fluent method: `.WithOutlierMode(mode)`; CLI: `--outlier <mode>`.

### OutlierDetector — default `null` (uses `OutlierMode`)

A custom `IOutlierDetector` strategy. When set, it takes precedence over `OutlierMode`, letting you plug in your own trimming algorithm. Built-in detector instances are available via `OutlierDetectors` (`None`, `RemoveTop5Percent`, `RemoveTopAndBottom5Percent`, `IqrFence`, `MedianAbsoluteDeviation`) and `OutlierDetectors.ForMode(mode)`. The detector actually used is reported in `BenchmarkResult.OutlierDetector`. Suite fluent: `.WithOutlierDetector(detector)`.

### TailMetricsBasis — default `TailMetricsBasis.Raw`

Which sample set the order statistics (percentiles, min, max, histogram) are computed from. `Raw` (the default) — the full pre-trim distribution, so tail metrics describe the tail the outlier fence removed rather than the inliers. `Trimmed` — only the kept samples. Central-tendency and dispersion statistics always stay on the trimmed set.

### ConfidenceLevel — default `0.95`, range `>0` and `<1`

Confidence level for the margin of error on the mean (the Error column). `0.90` is narrower/less conservative; `0.99` is wider/more conservative. A higher level produces a larger Error. Suite: `.WithConfidenceLevel(0.99)`; CLI: `--confidence 0.99`.

The margin is `StudentT.CriticalValue(confidenceLevel, n−1) × StandardError`, where `StandardError = StandardDeviation / √n`. The true mean lies within `Mean ± MarginOfError` at the given level. A separate distribution-free CI on the median is populated in `BenchmarkResult.MedianCiLower` / `MedianCiUpper`.

### ReportedPercentiles — default `[0.50, 0.95, 0.99, 0.999, 1.0]`

The set of percentiles to compute and report (values in [0, 1]). `1.0` reports the sample maximum. Values are normalized to ascending order with duplicates removed. Exposed on `BenchmarkResult.Percentiles` as `IReadOnlyList<PercentileEntry>`; use `GetPercentile(p)` for convenient lookup.

### EnableHistogram — default `true`

Whether to compute a latency histogram from the trimmed samples. Set to `false` to skip histogram computation and keep `BenchmarkResult.Histogram` null.

### HistogramBucketCount — default `20`, range `5`-`100`

The number of buckets in the latency histogram (only used when `EnableHistogram` is true).

### EnableSignificance — default `true`

When `true` and there are ≥2 benchmarks, runs a significance test against the baseline. Disable with `.WithSignificance(false)` to skip the overhead.

### SignificanceTest — default `null` (uses `DefaultSignificanceTest`)

A custom `ISignificanceTest` strategy. When set, it takes precedence over the built-in default (Mann-Whitney U for two groups, Kruskal-Wallis for three or more). Built-in strategies: `DefaultSignificanceTest`, `MannWhitneyUSignificanceTest`, `KruskalWallisSignificanceTest`. Suite fluent: `.WithSignificanceTest(test)`.

### SignificanceLevel — default `0.05`, range `>0` and `<1`

The alpha threshold a p-value must fall below to be reported as **Significant**. Lower it (e.g. `0.01`) to demand stronger evidence before calling a difference real. This is independent of `ConfidenceLevel` (which only affects the Error column). Suite: `.WithSignificanceLevel(0.01)`; CLI: `--alpha 0.01`.

### MinimumPracticalEffect — default `0.147`, range `[0, 1]` or `null`

The minimum practical effect in [0, 1] required for a benchmark to be considered meaningfully different. The active significance strategy maps its own effect metric to this normalized value via `EffectSize.PracticalValue`. When the reported practical value is below this threshold, the Sig verdict is downgraded to `NotSignificant` and the magnitude label is forced to `neg`, and a warning records the downgrade. The default `0.147` is a "small" Cliff's δ, so a `✓` means "real and at least a small effect". Set to `0` to restore p-value-only Sig semantics; set to `null` to disable the gate entirely. Suite/Harness: `.WithMinimumPracticalEffect(delta)`.

### LaunchCount — not on `MeasurementOptions`

The replicate count - how many separate worker processes measure a benchmark - is **not** a field on `MeasurementOptions`. A replicate is a worker, and `MeasurementOptions` is serialized whole into each worker's request; a worker has no use for a count it is told to repeat, since it measures exactly once. The count lives on the `LaunchCounts` static class and is passed explicitly by whichever coordinator spends it.

| `LaunchCounts` member | Value / signature | Purpose |
|---|---|---|
| `Single` | `1` | One launch - measure once, in one process. Single/Suite default. |
| `Max` | `100` | The ceiling on any requested launch count. |
| `HarnessDefault` | `3` | What Harness mode launches when the caller pinned nothing. Above one so the cross-launch interval surfaces without users asking. |
| `IsValid` | `bool IsValid(int count)` | `count is >= Single and <= Max`. |
| `Clamp` | `int Clamp(int count)` | Brings `count` into range rather than rejecting it (for attribute paths, where the value is a compile-time constant). |

Set it with `WithLaunchCount(n)` (suite/harness), `--launch-count <n>` (CLI), `[Benchmark(LaunchCount = n)]` (harness attribute), or `LaunchCount` on the test-integration attributes. Higher values trigger per-launch aggregation and populate `BenchmarkResult.LaunchStatistics` (per-launch medians, mean, stddev, CI, and the reproducibility diagnostics `ProcessVarianceRatio` / `BetweenLaunchDispersion`).

### MaxRawSamples — default `4096` (`MeasurementOptions.DefaultMaxRawSamples`), range `0` or positive

How many raw samples an isolated worker returns per benchmark. `MeasurementOptions.UnboundedRawSamples` (0) returns every sample; any positive value bounds the returned array. CLI: `--emit-raw` lifts the cap entirely (equivalent to `UnboundedRawSamples`).

This bounds only what crosses a process boundary. Every statistic NBenchmark reports (median, interval, outlier count) is computed inside the worker over the complete sample array, so raising or lowering this cannot move a reported number. It affects only the sample dump in JSON output, the Console density sparkline, and the coordinator-side significance test - all distribution properties, which a few thousand samples describe as faithfully as a hundred thousand.

The subset is drawn uniformly at random from the full array and kept in measurement order, seeded from the run's own seed so a repeat of the same configuration ships the same samples. It is not a prefix: the first n samples are the part of the run nearest to warmup, which is the least representative slice available. In-process runs are unaffected - there is no boundary to cross, so they always hold the complete array.

### StreamSamples — default `false`

Whether an isolated worker forwards its live per-sample observer stream (`IMeasurementObserver.OnSample`) back to the coordinator. Off by default. CLI: `--stream-samples`.

Like `MaxRawSamples` this bounds only what crosses a process boundary and cannot move a reported number. It is off by default because it is the one channel whose cost scales with how fast the benchmarked code is: a nanosecond body emits thousands of sample events, and encoding them puts the cost of observing the run inside the run. Phase transitions, detector snapshots, and results cross either way - they are emitted a handful of times per benchmark. In-process runs ignore this: the observer is called directly, so there is no boundary to forward across. See [observers.md](../../nbenchmark-reporters/references/observers.md#isolated-worker-delivery).

### SuppressPerClassIndependenceWarning — default `false`

When `false` (the default), a runtime warning is emitted when a class with `InstanceLifetime.PerClass` has more than one `[Benchmark]` method, because shared state across methods violates the statistical-independence assumption of the significance test. Set to `true` to suppress this warning when sharing is intentional.

### Environment — default `null`

Opt-in hardware/OS controls applied for the duration of a run: `CpuAffinity` (`IReadOnlyList<int>?`), `ProcessPriority` (`ProcessPriorityClass?`), `DedicatedHostGuidance` (`bool`). `null` (the default) does nothing — the benchmark runs with whatever affinity and priority the host started it with. Set via `.WithHardwareAffinity(...)`, `.WithProcessPriority(...)`, `.WithDedicatedHostGuidance(...)`, the `--cpu-affinity` / `--priority` / `--dedicated-host-guidance` CLI flags, or directly on the options record.

## Per-method overrides (Harness mode)

In Harness mode, `[Benchmark(Iterations = 1000, WarmupIterations = 100)]` overrides the harness-level options for that one method. Internally these use `-1` as the "unset" sentinel (range otherwise 0-100,000 / 0-10,000). See the `nbenchmark-host` skill.

Only `Iterations` and `WarmupIterations` are pinnable per method. `OpsPerSample` is **not** exposed on `[Benchmark]` — pin it suite/host-wide with `.WithOpsPerSample(n)` or `--ops-per-sample n`.

## Dry run

`Iterations = 0` is the dry-run signal: the measured loop is skipped and a zeroed result is returned. With the default (auto) or `0` warmup the body is never invoked; a pinned positive `WarmupIterations` still runs that many warmup invocations first. CLI `--dry-run` is exactly `--iterations 0 --warmup 0`. To invoke the body once as a smoke test, use `--iterations 1 --warmup 0` instead.