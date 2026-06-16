# BenchmarkResult Reference

`BenchmarkResult` is an immutable `record` (namespace `NBenchmark`) produced by every mode. All times are in **nanoseconds** unless noted. Access any field directly:

```csharp
Console.WriteLine($"{result.Name}: {result.Median} ns median");
if (result.MeanAllocatedBytes is { } bytes)
    Console.WriteLine($"{bytes} B/op");
```

## Identity

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Benchmark name |
| `Description` | `string?` | Optional description (from `[Benchmark(Description = ...)]`) |
| `IsBaseline` | `bool` | Whether this result is the suite baseline |
| `RunAtUtc` | `DateTimeOffset` | When the benchmark ran |

## Central tendency & spread

| Field | Type | Description |
|---|---|---|
| `Mean` | `double` | Arithmetic mean after outlier trimming |
| `Median` | `double` | 50th percentile — the primary metric (robust to outliers) |
| `P95` / `P99` | `double` | 95th / 99th percentile (nearest-rank) |
| `Min` / `Max` | `double` | Smallest / largest measured sample |
| `StandardDeviation` | `double` | Sample standard deviation (Bessel's correction, n−1) |
| `StandardError` | `double` | `StandardDeviation / √n` |
| `CoefficientOfVariation` | `double` | `StandardDeviation / Mean` (0 when mean is 0) |

## Distribution shape

| Field | Type | Description |
|---|---|---|
| `Q1` / `Q3` | `double` | First / third quartile |
| `InterquartileRange` | `double` | `Q3 − Q1` |
| `LowerFence` / `UpperFence` | `double?` | IQR fences; populated only when `OutlierMode = IqrFence` |
| `Skewness` | `double` | Sample skewness (0 for n < 3) |
| `Kurtosis` | `double` | Excess kurtosis (0 for n < 4) |
| `Mad` | `double` | Median absolute deviation, scaled by 1.4826 |
| `OutliersRemoved` | `int` | Samples discarded by outlier trimming |
| `N` | `int` | Post-trim sample count |

## Confidence interval

| Field | Type | Description |
|---|---|---|
| `MarginOfError` | `double` | Half-width of the CI on the mean (the "Error" column) |
| `ConfidenceLevel` | `double` | Confidence level used (default 0.95) |
| `ConfidenceIntervalLower` | `double` (computed) | `Mean − MarginOfError` |
| `ConfidenceIntervalUpper` | `double` (computed) | `Mean + MarginOfError` |

## Allocations

| Field | Type | Description |
|---|---|---|
| `MeanAllocatedBytes` | `long?` | Mean bytes allocated per iteration (null if not measured) |
| `AllocMedian` | `long?` | Median allocation per iteration |
| `AllocP95` | `long?` | 95th-percentile allocation per iteration |
| `AllocMax` | `long?` | Max allocation per iteration |

Allocation fields are populated only when `MeasureAllocations = true`.

## Significance

| Field | Type | Description |
|---|---|---|
| `PValue` | `double?` | Mann-Whitney U p-value vs baseline (null if not tested) |
| `SignificanceVerdict` | `SignificanceVerdict` | `NotTested`, `Significant`, or `NotSignificant` |
| `SignificanceLevel` | `double` | Alpha threshold used (default 0.05) |

## Status & timing

| Field | Type | Description |
|---|---|---|
| `Errored` | `bool` | Whether the benchmark threw |
| `ErrorMessage` | `string?` | Exception message if errored |
| `MeasuredIterations` | `int` | Measured samples kept after outlier trim (auto-resolved unless pinned) |
| `WarmupIterations` | `int` | Warmup samples performed (auto-resolved unless pinned) |
| `TotalDuration` | `TimeSpan` | End-to-end wall clock incl. warmup + pre-measure GC + measured loop |
| `MeasuredDuration` | `TimeSpan` | Measured loop only (incl. per-iteration setup/teardown/GC) |
| `OutlierMode` | `OutlierMode` | Outlier strategy applied |
| `Warnings` | `IReadOnlyList<string>` | Non-fatal notes (e.g. bimodal-distribution warning) |

Contract: `MeasuredDuration <= TotalDuration`. Errored entries from pre-runner failures (e.g. suite setup) stay at `TimeSpan.Zero`.

## Adaptive tuning

| Field | Type | Description |
|---|---|---|
| `AutoTune` | `AutoTuneDiagnostic?` | What the adaptive loop resolved; `null` on dry-run and errored results |

`AutoTuneDiagnostic` records `ResolvedWarmup`, `ResolvedSamples`, `OpsPerSample` (K), `TotalBodyInvocations`, `WarmupStop` (`WarmupStopReason`: `Settled` / `MaxCeiling` / `ExplicitCount` / `WallClockCap`), `SampleStop` (`SampleStopReason`: `CiTargetMet` / `MaxCeiling` / `ExplicitCount` / `WallClockCap`), `AchievedRelativeCiWidth`, and `TuningWallClock`. The `MeasuredIterations` and `WarmupIterations` fields above report the same resolved counts (after any outlier trim).

## Computed percentage helpers

| Property | Formula |
|---|---|
| `Range` | `Max − Min` |
| `StandardErrorPercent` | `StandardError / Mean × 100` |
| `MarginPercent` | `MarginOfError / Mean × 100` |
| `CoefficientOfVariationPercent` | `CoefficientOfVariation × 100` |

## Related enums

```csharp
public enum SignificanceVerdict { NotTested, Significant, NotSignificant }
public enum OutlierMode { None, RemoveTop5Percent, RemoveTopAndBottom5Percent, IqrFence }
public enum RunOrder { Random, Declaration }
```
