# BenchmarkResult Reference

`BenchmarkResult` is an immutable `record` (namespace `NBenchmark`) produced by every mode. All times are in **nanoseconds** unless noted. Access any field directly:

```csharp
Console.WriteLine($"{result.Name}: {result.Median} ns median");
if (result.MeanAllocatedBytes is { } bytes)
    Console.WriteLine($"{bytes} B/op");

// Percentiles are a list (configurable via MeasurementOptions.ReportedPercentiles):
double? p95 = result.GetPercentile(0.95);
```

## Identity

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Benchmark name (required) |
| `ClassName` | `string` | Class that declared the benchmark; empty for suite-mode entries added directly |
| `Description` | `string?` | Optional description (from `[Benchmark(Description = ...)]`) |
| `Categories` | `IReadOnlyList<string>` | Categories tagged on the method/class |
| `ParameterSet` | `IReadOnlyList<BenchmarkParameter>` | Parameter values for parameterised benchmarks; empty otherwise |
| `RuntimeMoniker` | `string` | Runtime the benchmark ran under (e.g. `"Net10"`); empty for in-process runs |
| `RuntimeProfileName` | `string` | The runtime-startup configuration this result was **actually** measured under - not the one requested. `"host"` means the measurement ran in a process NBenchmark did not launch, so it inherited whatever runtime config that process started with; every in-process result reports this. Results under different profiles are never compared. Default `"host"`. See [isolation.md](isolation.md#runtime-profiles). |
| `RuntimeKnobs` | `string` | The runtime-startup knobs in effect, e.g. `"tiered=off pgo=off r2r=off"`. Read from the measuring process's own environment rather than derived from the requested profile. Empty when none are set. |
| `IsolationStatus` | `IsolationStatus` | Where this measurement ran, and - when it did not run in a worker - why not. Every result starts at `InProcessRequested` (the **runtime** default the runner stamps so an un-marked result never silently claims to be isolated) and is re-stamped to `Isolated` only when a worker actually launched. See [isolation.md](isolation.md#isolationstatus) for the values, labels, causes, and remedies. |
| `IsBaseline` | `bool` | Whether this result is the suite baseline |
| `RunAtUtc` | `DateTimeOffset` | When the benchmark ran |

## Central tendency & spread

| Field | Type | Description |
|---|---|---|
| `Mean` | `double` | Arithmetic mean after outlier trimming (required) |
| `Median` | `double` | 50th percentile — the primary metric (robust to outliers) (required) |
| `Min` / `Max` | `double` | Smallest / largest measured sample (required) |
| `Percentiles` | `IReadOnlyList<PercentileEntry>` | Configurable percentile values; default set is P50/P95/P99/P99.9/Max. Each entry is `(double Percentile, double Value)` sorted ascending. Use `GetPercentile(p)` for convenience. |
| `StandardDeviation` | `double` | Sample standard deviation (Bessel's correction, n−1) (required) |
| `StandardError` | `double` | `StandardDeviation / √n` |
| `CoefficientOfVariation` | `double` | `StandardDeviation / Mean` (0 when mean is 0) |
| `OperationsPerSecond` | `double` | `1e9 / Mean` (NaN for errored / dry-run) |
| `MedianOperationsPerSecond` | `double` | `1e9 / Median` |
| `NanosecondsPerOperation` | `double` | Convenience alias for `Mean` |

## Distribution shape

| Field | Type | Description |
|---|---|---|
| `Q1` / `Q3` | `double` | First / third quartile (required) |
| `InterquartileRange` | `double` | `Q3 − Q1` (required) |
| `LowerFence` / `UpperFence` | `double?` | IQR fences; populated only when `OutlierMode = IqrFence` |
| `Skewness` | `double` | Sample skewness (0 for n < 3) (required) |
| `Kurtosis` | `double` | Excess kurtosis (0 for n < 4) (required) |
| `Mad` | `double` | Median absolute deviation, scaled by 1.4826 (required) |
| `OutliersRemoved` | `int` | Samples discarded by outlier trimming (required) |
| `N` | `int` | Post-trim sample count (required) |
| `TrimmedOrdinals` | `IReadOnlyList<int>` | Zero-based positions in the raw-sample stream of every discarded sample, sorted ascending by value; empty for dry-run / errored / calibration results |
| `Histogram` | `LatencyHistogram?` | Latency histogram of trimmed samples (`Buckets`, `Min`, `Max`, `SampleCount`); `null` when `EnableHistogram = false` or <2 samples |

## Confidence interval (mean)

| Field | Type | Description |
|---|---|---|
| `MarginOfError` | `double` | Half-width of the CI on the mean (the "Error" column) |
| `ConfidenceLevel` | `double` | Confidence level used (default 0.95) |
| `ConfidenceIntervalLower` | `double` (computed) | `Mean − MarginOfError` |
| `ConfidenceIntervalUpper` | `double` (computed) | `Mean + MarginOfError` |

## Confidence interval (median)

| Field | Type | Description |
|---|---|---|
| `MedianCiLower` / `MedianCiUpper` | `double?` | Distribution-free (order-statistic) CI on the median; `null` for dry-run / errored / calibration / <2 samples |
| `MedianShift` | `ShiftEstimate?` | Hodges-Lehmann shift vs the baseline (median of pairwise candidate − baseline differences) with a rank-based CI, in ns/op; `null` for baseline / single-benchmark / not-run |

## Allocations

| Field | Type | Description |
|---|---|---|
| `MeanAllocatedBytes` | `long?` | Mean bytes allocated per iteration (null if not measured) |
| `AllocMedian` | `long?` | Median allocation per iteration (required; null if not measured) |
| `AllocP95` | `long?` | 95th-percentile allocation per iteration (required; null if not measured) |
| `AllocMax` | `long?` | Max allocation per iteration (required; null if not measured) |

Allocation fields are populated only when `MeasureAllocations = true` (the default).

## Significance

| Field | Type | Description |
|---|---|---|
| `PValue` | `double?` | p-value vs baseline (null if not tested) |
| `SignificanceVerdict` | `SignificanceVerdict` | `NotTested`, `Significant`, or `NotSignificant` |
| `Effect` | `EffectSize?` | Effect-size payload (Cliff's δ by default): `Metric`, `Value`, `Magnitude`, `Direction`, `PracticalValue` |
| `Omnibus` | `OmnibusComparison?` | Kruskal-Wallis omnibus verdict for 3+ groups: `TestName`, `Statistic`, `PValue`, `DegreesOfFreedom`, `GroupCount`, `Verdict` |
| `SignificanceTestName` | `string` | Name of the significance strategy (default `"Mann-Whitney U"` or `"Kruskal-Wallis"`) |
| `SignificanceLevel` | `double` | Alpha threshold used (default 0.05) |

The `MinimumPracticalEffect` gate (default 0.147 - a "small" Cliff's δ) downgrades a statistically-significant but practically-negligible result to `NotSignificant` and forces the magnitude label to `neg`.

## Status & timing

| Field | Type | Description |
|---|---|---|
| `Errored` | `bool` | Whether the benchmark threw |
| `ErrorMessage` | `string?` | Exception message if errored |
| `MeasuredIterations` | `int` | Measured samples kept after outlier trim (auto-resolved unless pinned) |
| `WarmupIterations` | `int` | Warmup samples performed (auto-resolved unless pinned) |
| `TotalOperations` | `long` | Total body invocations across warmup and measurement |
| `TotalDuration` | `TimeSpan` | End-to-end wall clock incl. warmup + pre-measure GC + measured loop |
| `MeasuredDuration` | `TimeSpan` | Measured loop only (incl. per-iteration setup/teardown/GC) |
| `OutlierMode` | `OutlierMode` | Outlier strategy applied |
| `OutlierDetector` | `string` | Name of the outlier detector used |
| `Profile` | `MeasurementProfile` | `Realistic` or `Independent` |
| `Warnings` | `IReadOnlyList<string>` | Non-fatal notes (e.g. bimodal-distribution warning, PerClass independence warning) |

Contract: `MeasuredDuration <= TotalDuration`. Errored entries from pre-runner failures (e.g. suite setup) stay at `TimeSpan.Zero`.

## Adaptive tuning

| Field | Type | Description |
|---|---|---|
| `AutoTune` | `AutoTuneDiagnostic?` | What the adaptive loop resolved; `null` on dry-run and errored results |

`AutoTuneDiagnostic` records `ResolvedWarmup`, `ResolvedSamples`, `OpsPerSample` (K), `InitialOpsPerSample`, `TotalBodyInvocations`, `WarmupStop` (`WarmupStopReason`: `Settled` / `MaxCeiling` / `ExplicitCount` / `WallClockCap`), `SampleStop` (`SampleStopReason`: `CiTargetMet` / `MaxCeiling` / `ExplicitCount` / `WallClockCap` / `GraceCapExhausted`), `AchievedRelativeCiWidth`, `TuningWallClock`, `JitterMetric`, and `OutlierDetectorSwitched`. The `MeasuredIterations` and `WarmupIterations` fields above report the same resolved counts (after any outlier trim).

## Multi-launch

| Field | Type | Description |
|---|---|---|
| `LaunchStatistics` | `LaunchStatistics?` | Populated when `LaunchCount > 1`; `null` otherwise |

`LaunchStatistics` records `LaunchCount`, `LaunchMean`, `LaunchStandardDeviation`, `LaunchMedian`, `LaunchConfidenceIntervalLower` / `Upper`, `BetweenLaunchDispersion`, `ProcessVarianceRatio`, and `Launches` (`IReadOnlyList<LaunchDetail>`). Each `LaunchDetail` has `LaunchIndex`, `Median`, `Mean`, `StandardDeviation`, `Iterations`, `Duration`, `Errored`, `ErrorMessage`.

`BetweenLaunchDispersion` is the coefficient of variation of the per-launch medians - run-to-run variation as a fraction of the typical measurement (the **reproducibility** of the number, as opposed to the precision of any one launch). `null` when fewer than two launches succeeded.

`ProcessVarianceRatio` is how much larger the spread **between** processes is than the spread **within** one. Near 1 means the within-process confidence interval fairly describes what a re-run would produce; a large value means it does not. This exposes the most dangerous failure mode in benchmarking: a tight interval around a value that does not reproduce. `null` when fewer than two launches succeeded.

## Diagnostics

| Field | Type | Description |
|---|---|---|
| `Diagnostics` | `DiagnosticsResult?` | Runtime counters when `Diagnostics` options enabled; `null` otherwise |

`DiagnosticsResult` records `Gen0Collections`, `Gen1Collections`, `Gen2Collections`, `HeapCommittedBytes`, `HeapFragmentedBytes`, `ExceptionCountPerOp`, `CpuTimeNsPerOp`, `CpuWallRatio`, and `Mode` (`DiagnosticsMode`).

## Computed percentage helpers

| Property | Formula |
|---|---|
| `Range` | `Max − Min` |
| `StandardErrorPercent` | `StandardError / Mean × 100` |
| `MarginPercent` | `MarginOfError / Mean × 100` |
| `CoefficientOfVariationPercent` | `CoefficientOfVariation × 100` |

## Public methods

| Method | Signature | Description |
|---|---|---|
| `GetPercentile` | `double? GetPercentile(double p)` | Linear scan over `Percentiles`; returns `null` if `p` isn't in the list |
| `FromCalibration` (static) | `BenchmarkResult FromCalibration(string name, double mean, double median, double[] samples)` | Build a calibration-style result (no auto-tune / no diagnostics) |

## Related enums

```csharp
public enum SignificanceVerdict { NotTested, Significant, NotSignificant }
public enum IsolationStatus { Isolated = 0, InProcessRequested, InProcessCapturedState, InProcessLiveFixture, InProcessUnaddressablePlan, InProcessNoWorker }
public enum OutlierMode { None, RemoveTop5Percent, RemoveTopAndBottom5Percent, IqrFence, MedianAbsoluteDeviation }
public enum RunOrder { Random, Declaration }
public enum MeasurementProfile { Realistic, Independent }
public enum TailMetricsBasis { Raw, Trimmed }
public enum RuntimeMoniker { Net8, Net9, Net10 }
public enum AutoTunePreset { Default, Quick, Thorough }
public enum AutoTuneCapBehavior { Warn, Error }
public enum DiagnosticsMode { None = 0, GcCollectionCounts = 1, GcHeapInfo = 2, Exceptions = 4, CpuTime = 8,
                               Gc = GcCollectionCounts | GcHeapInfo, GcAndCpu = Gc | CpuTime, All = Gc | Exceptions | CpuTime }
public enum SampleStopReason { CiTargetMet, MaxCeiling, ExplicitCount, WallClockCap, GraceCapExhausted }
public enum WarmupStopReason { Settled, MaxCeiling, ExplicitCount, WallClockCap }
```