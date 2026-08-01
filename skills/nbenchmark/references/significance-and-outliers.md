# Significance & Outliers Reference

NBenchmark's statistics pipeline lives in `NBenchmark.Stats` (with a few types in `NBenchmark` and `NBenchmark.Engine`). This reference covers the pluggable significance tests, outlier detectors, effect-size payloads, and the warning strings that surface on `BenchmarkResult.Warnings`.

## How it fits together

Every measured benchmark flows through one pipeline:

```
raw timings  ->  OutlierTrim.TrimDetailed  ->  StatsSummary.Compute  ->  SampleQuality.BuildWarnings
                     |                                |                       |
                     v                                |                       |
              ProcessedMeasurements ------------------+-----------------------+
                                               |
                                               v
                          Significance.ApplyIfEnabled  (vs baseline)
                                               |
                                               v
                                       BenchmarkResult
```

- `OutlierTrim` splits the raw stream into kept/discarded using the configured `IOutlierDetector`.
- `StatsSummary.Compute` produces the central-tendency, dispersion, percentile, histogram, and median-CI stats.
- `SampleQuality.BuildWarnings` runs post-hoc i.i.d. sanity checks on the arrival-order stream.
- `Significance.ApplyIfEnabled` runs the `ISignificanceTest` against the baseline and assigns `PValue` / `SignificanceVerdict` / `Effect` / `MedianShift` / `Omnibus` to each non-baseline result.

You usually don't call these directly - `MeasurementOptions` exposes the knobs. This reference is for plugging in custom strategies or deeply interpreting results.

## Significance testing

### `ISignificanceTest`

The strategy interface. Implement this to plug in a custom test.

```csharp
namespace NBenchmark.Stats;

public interface ISignificanceTest
{
    string Name { get; }
    SignificanceReport Analyze(SignificanceContext context);
}
```

`SignificanceContext` is a record carrying the sample groups, the baseline index, and alpha:

| Property | Type | Description |
|---|---|---|
| `Groups` | `IReadOnlyList<SampleGroup>` | All groups (baseline + candidates) |
| `BaselineIndex` | `int` | Index of the baseline in `Groups` |
| `SignificanceLevel` | `double` | Alpha threshold |
| `Baseline` | `SampleGroup` | Computed: `Groups[BaselineIndex]` |
| `Candidates` | `IEnumerable<SampleGroup>` | Computed: every non-baseline group |

`SampleGroup` is a readonly record struct: `SampleGroup(string Name, double[] Samples, bool IsBaseline)`.

`SignificanceReport` is a record returned by `Analyze`:

| Property | Type | Description |
|---|---|---|
| `Pairwise` | `IReadOnlyList<PairwiseComparison>` | Per-candidate verdicts |
| `Omnibus` | `OmnibusComparison?` | Optional omnibus verdict (3+ groups) |
| `Empty` (static) | `SignificanceReport` | `{ Pairwise = [] }` - empty report |

`PairwiseComparison` is a readonly record struct: `(string Name, double? PValue, SignificanceVerdict Verdict, EffectSize? Effect = null, ShiftEstimate? Shift = null)`.

### Built-in strategies

| Type | Singleton | `Name` | When used |
|---|---|---|---|
| `DefaultSignificanceTest` | `Instance` | `"Mann-Whitney U"` | The default. Selects strategy by candidate count. |
| `MannWhitneyUSignificanceTest` | `Instance` | `"Mann-Whitney U"` | 2 groups (baseline + 1 candidate). |
| `KruskalWallisSignificanceTest` | `Instance` | `"Kruskal-Wallis"` | 3+ groups (omnibus). |

`DefaultSignificanceTest` picks the strategy:
- **2 groups** (1 candidate) - delegates to `MannWhitneyUSignificanceTest`. Returns pairwise only, no omnibus.
- **3+ groups** - runs `KruskalWallisSignificanceTest` first. If the omnibus verdict is not significant, returns the omnibus-only report. If significant, runs pairwise Mann-Whitney U per candidate vs baseline, applies `MultipleComparisons.HolmBonferroni` over the raw p-values, builds a `PairwiseComparison` per candidate (with Cliff's δ `EffectSize` and Hodges-Lehmann `ShiftEstimate`), and returns a `SignificanceReport` carrying both pairwise verdicts and the original omnibus.

All three are stateless singletons. To plug in your own, implement `ISignificanceTest` and set it via `MeasurementOptions.SignificanceTest` (or `.WithSignificanceTest(test)`).

### Low-level stats utilities

These are used internally and exposed for direct use:

| Type | Key member | What it does |
|---|---|---|
| `MannWhitneyU` (static) | `MannWhitneyUResult Test(double[] sampleA, double[] sampleB)` | Two-sided Mann-Whitney U. Returns `NaN` p-value when either group has <2 samples. `ExactMaxCombinedSamples = 20` (exact permutation below, asymptotic normal approximation above). |
| `KruskalWallis` (static) | `KruskalWallisResult Test(IReadOnlyList<double[]> groups)` | Omnibus H test. Returns `PValue = NaN` when <2 groups, any empty group, or <2 total observations. `MinGroups = 2`. |
| `HodgesLehmann` (static) | `ShiftEstimate? Estimate(double[] baseline, double[] candidate, double confidenceLevel)` | Hodges-Lehmann shift (median of pairwise differences) with rank-based CI. Returns `null` when either group has <2 samples. `MaxPerGroup = 512` (deterministic stride subsampling above). |
| `MultipleComparisons` (static) | `double[] HolmBonferroni(IReadOnlyList<double> rawPValues)` | Holm-Bonferroni family-wise error correction. NaN inputs preserved, non-NaN inputs step-up adjusted and capped at 1.0. |
| `MedianCi` (static) | `(double Lower, double Upper)? Compute(double[] sorted, double confidenceLevel)` | Distribution-free (order-statistic) CI for the median. Exact binomial search for n < 50, normal approximation for n >= 50. Returns `null` when n < 2. |
| `StudentT` (static) | `double CriticalValue(double confidenceLevel, int degreesOfFreedom)` | Two-tailed critical t. `InverseCdf(p, df)` and `NormalQuantile(p)` also available. |
| `ChiSquared` (static) | `double SurvivalFunction(double x, int degreesOfFreedom)` | `P(X > x)` for chi-squared with `df`. |
| `Percentile` (static) | `double Compute(double[] sorted, double p)` | Nearest-rank percentile (median p=0.50 uses mid-average on even n). |

### Result types

| Type | Properties |
|---|---|
| `MannWhitneyUResult` (readonly record struct) | `PValue`, `CliffsDelta` (positive = candidate slower) |
| `KruskalWallisResult` (readonly record struct) | `H`, `DegreesOfFreedom`, `PValue`, `GroupCount`, `IsValid` (computed) |
| `ShiftEstimate` (readonly record struct) | `Value`, `Lower`, `Upper`, `ConfidenceLevel` |
| `OmnibusComparison` (sealed record, namespace `NBenchmark`) | `TestName`, `Statistic`, `PValue`, `DegreesOfFreedom`, `GroupCount`, `Verdict` |
| `SignificanceVerdict` (enum, namespace `NBenchmark`) | `NotTested`, `Significant`, `NotSignificant` |

### Convenience methods

The `Significance` static class (namespace `NBenchmark.Stats`) provides high-level entry points used by the runner:

```csharp
// Apply significance if MeasurementOptions.EnableSignificance is true
Significance.ApplyIfEnabled(results, rawSamples, options);

// Or call directly with the default strategy
Significance.ComputeSignificance(results, rawSamples, significanceLevel: 0.05, minimumPracticalEffect: 0.147);

// Or with a custom strategy
Significance.ComputeSignificance(results, rawSamples, myTest, significanceLevel: 0.05, minimumPracticalEffect: 0.147);
```

`ComputeSignificance` mutates `results` in place: assigns `PValue`, `SignificanceVerdict`, `Effect`, `MedianShift`, `Omnibus`, and may append to `Warnings`.

## Paired ratio estimation

The plain `Ratio` column divides two aggregated medians and can say nothing about how much that number would move on a re-run. When the launch count (see the `LaunchCounts` static class) is **two or more**, NBenchmark computes a paired, log-scale ratio estimate instead, and `--threshold-pct` gates on it.

### Why paired

A comparison group is measured **co-resident in one worker per replicate**: replicate *i* of the candidate and replicate *i* of the baseline ran in the same process, on the same core draw, under the same thermal state and the same address-space layout. Dividing them cancels all of that out of the ratio. Dividing the two *aggregated* medians instead throws the pairing away and leaves every worker-to-worker difference in the numerator and denominator independently - which is the entire statistical reason the group is co-resident, discarded at the last step.

### Why the log scale

A ratio is multiplicative, and its sampling distribution is right-skewed: 2x slower and 2x faster are equally large effects but sit at +1.0 and -0.5 on a linear scale. Taking logs makes them symmetric (+0.69 and -0.69), which is what a Student-t interval assumes. Exponentiating back gives an interval that is multiplicatively symmetric about the estimate and cannot straddle zero - a linear interval on a ratio near 1.0 with real spread routinely produces a negative lower bound, which is not a ratio.

The point estimate is the **geometric** mean of the per-replicate ratios, which is *not* the ratio of the arithmetic means. That difference is the correction: the geometric mean is the unbiased estimator of a multiplicative effect, and it is unmoved by one replicate in which *both* benchmarks happened to run slowly.

### `RatioEstimate`

The estimate lands on `BenchmarkTable` rows (and on each `BenchmarkResult` via the table) as a `RatioEstimate` (namespace `NBenchmark`):

| Property | Type | Description |
|---|---|---|
| `Value` | `double` | The estimated ratio: the geometric mean of the per-replicate ratios. Above 1.0 means slower than the reference. |
| `Lower` / `Upper` | `double` | The multiplicative confidence interval on `Value`. Read as the range the ratio would plausibly land in if the whole run were repeated. |
| `Replicates` | `int` | How many replicates were paired to produce this. Always at least two. |
| `ConfidenceLevel` | `double` | The level `Lower`/`Upper` were computed at. |
| `IncludesUnity` | `bool` (computed) | Whether the interval contains 1.0 - whether "no difference" is among the values this run cannot rule out. |
| `FormatInterval()` | `string` | The interval rendered as a multiplicative range, e.g. `"1.08-1.31x"`. Empty when the bounds are not finite. |

An interval **spanning 1.0** means the two benchmarks are not distinguishable at this replicate count, however far `Value` is from 1.0 - which is the question a reader is actually asking when they look at a ratio, and the one a bare number cannot answer. Reporters surface inconclusive ratios (including a unity-spanning warning); CSV writes `RatioCiLower`/`RatioCiUpper`/`RatioReplicates` columns.

### `LogRatio`

The static estimator (namespace `NBenchmark.Stats`):

```csharp
public static RatioEstimate? Estimate(
    IReadOnlyList<double> candidate,   // per-replicate medians of the benchmark being measured
    IReadOnlyList<double> baseline,   // per-replicate medians of the reference, same replicate order
    double confidenceLevel = 0.95);
```

Returns `null` when fewer than two replicates can be paired. One pair is a ratio, not an estimate of one - it carries no information about how much the ratio would move on a re-run. Non-positive medians (an errored or unmeasured replicate) drop the whole pair to keep the comparison paired.

### What gates on it

- `--threshold-pct` evaluates the **paired** ratio when launch data is available, instead of a ratio of aggregated medians. A gate can therefore change verdict on unchanged code, because the estimate it reads changed.
- Cross-configuration ratios (e.g. tiered vs not, or different runtime profiles) stay withheld as `n/a` regardless of launch count - a paired estimate still describes two processes more than two code paths.
- A mixed-isolation ratio (one side isolated, one in-host) is never enforced; see [isolation.md](isolation.md#the-capture-fallback).

## Effect size

### `EffectSize`

A readonly record struct carrying the effect-size payload reported on `BenchmarkResult.Effect`:

| Property | Type | Description |
|---|---|---|
| `Metric` | `string` | Identifies the statistic (e.g. `"Cliff's δ"`) |
| `Value` | `double?` | The raw effect-size value |
| `Magnitude` | `string?` | Strategy-defined qualitative label (e.g. `"small"`, `"large"`, `"neg"`) |
| `Direction` | `EffectDirection` | `None`, `CandidateHigher`, or `CandidateLower` |
| `PracticalValue` | `double?` | Normalized [0, 1] value used by `MeasurementOptions.MinimumPracticalEffect` |

`EffectDirection` is an enum: `None = 0`, `CandidateHigher`, `CandidateLower`.

### Building Cliff's δ

`EffectSizeFactory.ForCliffsDelta(double cliffsDelta)` builds the built-in Cliff's δ payload:
- `Metric = EffectMetrics.CliffsDelta` (`"Cliff's δ"`)
- `Value = cliffsDelta`
- `Magnitude = MagnitudeLabelExtensions.Classify(|delta|).ToShortString()` (Romano thresholds)
- `Direction` based on sign (`CandidateHigher` if positive/slower, `CandidateLower` if negative/faster)
- `PracticalValue = |delta|`

### Magnitude labels

`MagnitudeLabel` enum (Romano 2006 thresholds on |δ|):

| Member | Threshold | `ToShortString()` |
|---|---|---|
| `Negligible` | |δ| < 0.147 | `"neg"` |
| `Small` | |δ| < 0.33 | `"small"` |
| `Medium` | |δ| < 0.474 | `"med"` |
| `Large` | |δ| >= 0.474 | `"large"` |

`MagnitudeLabelExtensions.Classify(double absDelta)` returns the `MagnitudeLabel` for a given absolute Cliff's δ. The `.ToShortString()` extension returns the lowercase abbreviation.

### Minimum practical effect gate

`MeasurementOptions.MinimumPracticalEffect` (default `0.147`, the boundary between `Negligible` and `Small`) downgrades a statistically-significant but practically-negligible result to `NotSignificant` and forces the `Magnitude` label to `"neg"`. A warning records the downgrade. Set to `0` for p-value-only semantics, or `null` to disable the gate entirely.

## Outlier detection

### `IOutlierDetector`

The strategy interface for trimming. Implement this to plug in a custom detector.

```csharp
namespace NBenchmark.Stats;

public interface IOutlierDetector
{
    string Name { get; }
    OutlierClassification Classify(double[] sortedSamples);
}
```

`Classify` receives the **sorted ascending** sample array (must not be mutated) and returns an `OutlierClassification`:
- `Kept` - inliers, sorted ascending
- `Discarded` - outliers, sorted ascending
- `LowerFence` / `UpperFence` - optional rejection boundaries (fence-based detectors only)
- `KeepAll(double[] sortedSamples)` (static) - convenience factory

If a rule would discard everything, return all samples unchanged.

### Built-in detectors

All live in `NBenchmark.Stats` and are returned by `OutlierDetectors.ForMode(mode)` or as static properties:

| Detector | Constructor | `Name` | Maps to `OutlierMode` |
|---|---|---|---|
| `NoOutlierDetector` | (none) | `"none"` | `None` |
| `TopPercentileOutlierDetector` | `(double fraction = 0.05)` - strictly (0, 1) | `"top 5%"` | `RemoveTop5Percent` |
| `TwoSidedPercentileOutlierDetector` | `(double fraction = 0.05)` - strictly (0, 0.5) | `"top & bottom 5%"` | `RemoveTopAndBottom5Percent` |
| `IqrFenceOutlierDetector` | `(double k = 1.5)` - k > 0 | `"IQR fence (1.5×)"` | `IqrFence` |
| `MadOutlierDetector` | `(double threshold = 3.0)` - threshold > 0 | `"MAD (3.0×)"` | `MedianAbsoluteDeviation` |

`OutlierDetectors` static factory:
- `OutlierDetectors.None` / `.RemoveTop5Percent` / `.RemoveTopAndBottom5Percent` / `.IqrFence` / `.MedianAbsoluteDeviation` - the built-in instances
- `OutlierDetectors.ForMode(OutlierMode mode)` - resolves the enum to its detector (default branch returns `IqrFence`)

### `OutlierMode` enum (namespace `NBenchmark`)

| Member | Behaviour |
|---|---|
| `None` | Keep all samples (sort only) |
| `RemoveTop5Percent` | Trim slowest 5% |
| `RemoveTopAndBottom5Percent` | Trim 5% from each end |
| `IqrFence` | Remove samples outside `Q1 - 1.5×IQR` ... `Q3 + 1.5×IQR` (default) |
| `MedianAbsoluteDeviation` | Remove samples outside `median ± 3.0 × scaledMAD` (scaledMAD = 1.4826 × median(|x - m|)) |

### `OutlierTrim` static class

Higher-level trimming that returns both kept samples and ordinals:

| Member | Signature |
|---|---|
| `Trim` | `double[] Trim(double[] timings, OutlierMode mode)` - kept samples only |
| `TrimDetailed` | `TrimResult TrimDetailed(double[] timings, OutlierMode mode)` |
| `TrimDetailed` | `TrimResult TrimDetailed(double[] timings, IOutlierDetector detector)` |

`TrimResult` (readonly record struct) carries: `Kept`, `Discarded`, `Q1`, `Q3`, `InterquartileRange`, `LowerFence`, `UpperFence`, `TrimmedOrdinals` (original positions of every discarded sample, in the same order as `Discarded`), `SortedAll` (the full pre-trim set handed to the detector).

### Customizing

Set `MeasurementOptions.OutlierDetector` to override `OutlierMode`. The detector actually used is reported in `BenchmarkResult.OutlierDetector`.

```csharp
var options = new MeasurementOptions
{
    OutlierDetector = new IqrFenceOutlierDetector(k: 3.0), // stricter fence
};
```

## Stats pipeline

### `StatsPipeline` (namespace `NBenchmark.Engine`)

The full trim -> summary -> warnings pipeline:

```csharp
ProcessedMeasurements Run(
    double[] rawTimings,
    long[]? rawAllocations,
    MeasurementOptions options,
    int[]? perSampleGcCounts = null);
```

Does not mutate `rawTimings`. Uses `options.ResolveOutlierDetector()`, selects the tail source per `options.TailMetricsBasis` (`Raw` = full pre-trim, `Trimmed` = inliers), computes `StatsSummary` via `StatsSummary.Compute`, and builds warnings.

### `ProcessedMeasurements` (namespace `NBenchmark.Engine`)

A sealed record with positional parameters `Stats`, `MeasuredIterations`, `MeanAllocatedBytes`, `Q1`, `Q3`, `InterquartileRange`, `LowerFence`, `UpperFence`, `OutliersRemoved`, `RawAllocations`, `TrimmedOrdinals`, plus init-only `Warnings` (`IReadOnlyList<string>`) and `DiagnosticsResult`.

### `StatsSummary` (namespace `NBenchmark.Stats`)

The descriptive-statistics block. Init-only properties: `Mean`, `Median`, `Percentiles`, `Histogram`, `Min`, `Max`, `StandardDeviation`, `StandardError`, `MarginOfError`, `ConfidenceLevel`, `CoefficientOfVariation`, `Skewness`, `Kurtosis`, `Mad`, `MedianCiLower`, `MedianCiUpper`.

Static methods:
- `StatsSummary.Compute(double[] samples, double confidenceLevel = 0.95, IReadOnlyList<double>? reportedPercentiles = null, bool enableHistogram = true, int histogramBucketCount = 20, double[]? tailSource = null)` - computes the full summary. When `tailSource` is supplied, percentiles/min/max/histogram read from it while central-tendency and dispersion stats stay on `samples`.
- `AllocationStats ComputeAllocations(long[]? samples)` - returns `AllocationStats(Mean, P50, P95, Max)`.

## Warnings

`BenchmarkResult.Warnings` collects non-fatal notes from several sites. The strings that flow into it:

### Bimodal distribution (from `StatsPipeline`)

When the discarded slow outliers form a tight secondary cluster (a possible second execution profile), `BimodalDetector.DetectSlowCluster` returns the cluster's count and centre. The warning reads:

> `"{count} discarded outlier(s) form a distinct cluster near {center} rather than scattered noise - possible bimodal distribution; investigate this tail latency (e.g. GC pauses, lock contention, or cache misses)."`

When some of those discarded outliers also coincided with a GC, a suffix is appended:

> `" ({gcCorrelatedOutliers} of the discarded outliers coincided with a garbage collection.)"`

### GC-correlated outliers without bimodal cluster (from `StatsPipeline`)

> `"{gcCorrelatedOutliers} of {removed} removed outlier(s) coincided with a garbage collection."`

### Sample-quality drift (from `SampleQuality.BuildWarnings`)

Post-hoc i.i.d. sanity checks on the arrival-order stream (skipped below `MinSamplesForChecks = 50` samples). When a split-half Mann-Whitney p-value falls below `DriftPValueThreshold = 0.001`:

> `"the first and second halves of the measured stream differ significantly (split-half Mann-Whitney p = {p}) - the timings drifted during measurement (JIT tier-up/DPGO, thermal ramp, or periodic GC), so the reported confidence interval may understate the true uncertainty; consider a longer warmup (--min-warmup-time) or checking host thermal/load state."`

### Sample-quality autocorrelation (from `SampleQuality.BuildWarnings`)

When the lag-1 autocorrelation exceeds `AutocorrelationThreshold = 0.5`:

> `"consecutive samples are correlated (lag-1 autocorrelation r = {r:F2}) - the samples are not independent, so the confidence interval understates uncertainty (effective sample size ≈ {effectiveN:F0} of {n})."`

### Practical-effect downgrade (from `Significance.ApplyReport`)

When a statistically-significant result is downgraded by the `MinimumPracticalEffect` gate:

> `"statistically significant but practically negligible: {metric} practical magnitude {practical:0.###} is below the minimum practical effect {minimumPracticalEffect:0.###}, so the significance verdict was downgraded to not-significant. Set MinimumPracticalEffect = 0 (CLI: --min-practical-effect 0) to restore p-value-only verdicts."`

### Significance error paths (from `Significance.AppendWarning`)

> `"No raw samples were captured for '{result.Name}', so it was excluded from significance testing."`
>
> `"Significance testing was skipped: fewer than two benchmarks had captured raw samples (baseline '{baseline.Name}')."`

Additional warnings come from `BenchmarkRunner.BuildMidBatchGcWarnings` and `AdaptiveLoop.BuildStopWarnings` (in `NBenchmark.Engine`, outside the Stats scope) and are merged into `BenchmarkResult.Warnings`.

## Related references

- [benchmark-result.md](benchmark-result.md) - where these stats land on `BenchmarkResult`
- [measurement-options.md](measurement-options.md) - the knobs that select the strategy
- The `nbenchmark-troubleshooting` skill - diagnosing noisy results and choosing outlier modes