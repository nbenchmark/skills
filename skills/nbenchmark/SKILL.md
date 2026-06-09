---
name: nbenchmark
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-09
description: Core NBenchmark library skill for writing .NET benchmarks using Quick mode (Benchmark.Run / RunAsync) and Suite mode (BenchmarkSuite fluent builder). Use when the user wants to benchmark a single method, compare multiple implementations, configure measurement options, or understand benchmark results.
---

# NBenchmark

NBenchmark is a lightweight, async-native .NET benchmarking library. Three modes exist: Quick (single call), Suite (fluent builder for A/B comparison), and Host (attribute-based CLI runner).

## When to use this skill

User asks to:

- Add benchmarking to a .NET project
- Benchmark a single method (`Benchmark.Run`)
- Compare two or more implementations (`BenchmarkSuite`)
- Configure iterations, warmup, outlier mode, or confidence level
- Understand what the output numbers mean
- Prevent dead-code elimination in benchmarks

## Quick Mode

Use `Benchmark.Run` / `Benchmark.RunAsync` for one-off measurements. Eight methods exist: `Run`, `Run<T>`, `RunAsync`, `RunAsync<T>`, `RunRaw`, `RunRaw<T>`, `RunRawAsync`, `RunRawAsync<T>`.

### Sync void

```csharp
BenchmarkResult result = Benchmark.Run(() => Array.Sort(data));
result.Print();
```

### Sync with return value (prevents dead-code elimination)

```csharp
BenchmarkResult result = Benchmark.Run(() => ComputeHash(data));
```

### Async void

```csharp
BenchmarkResult result = await Benchmark.RunAsync(async () => await FetchDataAsync());
```

### Async with return value

```csharp
BenchmarkResult result = await Benchmark.RunAsync(async () => await LoadFromDbAsync());
```

### Raw samples - access pre-trim timings

All four `RunRaw*` methods return `MeasurementOutcome` instead of `BenchmarkResult`, giving access to `RawSamples` (the full timing array before outlier trimming).

```csharp
// Sync raw
MeasurementOutcome outcome = Benchmark.RunRaw(() => MyMethod());
// Sync raw with return value
MeasurementOutcome outcome = Benchmark.RunRaw(() => ComputeHash(data));
// Async raw
MeasurementOutcome outcome = await Benchmark.RunRawAsync(async () => await MyMethodAsync());
// Async raw with return value
MeasurementOutcome outcome = await Benchmark.RunRawAsync(async () => await ComputeHashAsync(data));

double[] rawTimings = outcome.RawSamples;
BenchmarkResult result = outcome.Result;
```

### Custom options

```csharp
var result = Benchmark.Run(() => MyMethod(),
    options: new MeasurementOptions
    {
        Iterations = 500,
        WarmupIterations = 50,
        ConfidenceLevel = 0.99,
        MeasureAllocations = true,
        OutlierMode = OutlierMode.IqrFence,
    });
```

### Custom name

```csharp
var result = Benchmark.Run(() => MyMethod(), name: "MyMethod-v2");
```

### Single-result file output

```csharp
await result.ToMarkdownAsync("results.md");
await result.ToJsonAsync("./output");
await result.ToCsvAsync("results.csv");
await result.PrintAsync(); // requires NBenchmark.Console package
```

## Suite Mode

Use `BenchmarkSuite` to compare multiple implementations side-by-side with ratio and statistical significance.

### Basic comparison

```csharp
var results = await new BenchmarkSuite("Sorting")
    .Add("BubbleSort", () => BubbleSort(data))
    .Add("QuickSort", () => QuickSort(data))
    .WithBaseline("QuickSort")
    .RunAsync();
```

### Four Add overloads

```csharp
// Sync void
suite.Add("name", () => DoSomething());
// Sync returning (prevents dead-code elimination)
suite.Add("name", () => ComputeSomething());
// Async void
suite.Add("name", async () => await DoSomethingAsync());
// Async returning
suite.Add("name", async () => await ComputeSomethingAsync());
```

### Per-benchmark setup/teardown

```csharp
suite.Add("WithPrep", () => MeasuredMethod(),
    setup: () => PrepareData(),
    teardown: () => CleanupData());
```

### Fluent configuration methods

| Method                         | What it sets                                                 |
| ------------------------------ | ------------------------------------------------------------ |
| `WithBaseline(name)`           | Sets which benchmark is the reference for ratio/significance |
| `WithIterations(n)`            | Overrides `MeasurementOptions.Iterations`                    |
| `WithWarmup(n)`                | Overrides `MeasurementOptions.WarmupIterations`              |
| `WithAllocations(true)`        | Enables per-iteration allocation measurement                 |
| `WithOutlierMode(mode)`        | Sets outlier trimming strategy                               |
| `WithConfidenceLevel(level)`   | Sets confidence level (0-1, e.g. 0.99)                       |
| `WithSignificance(true/false)` | Enables/disables Mann-Whitney U significance test            |
| `WithRunOrder(order)`          | `RunOrder.Random` (default) or `RunOrder.Declaration`        |
| `WithSuiteSetup(action)`       | Action before all benchmarks                                 |
| `WithSuiteTeardown(action)`    | Action after all benchmarks                                  |
| `WithReporter(reporter)`       | Adds an `IReporter` for output                               |
| `WithProgress(progress)`       | Sets an `IBenchmarkProgress` callback                        |

### Full-featured example

```csharp
var results = await new BenchmarkSuite("JSON Parsers")
    .Add("System.Text.Json", () => SystemTextJson.Parse(data))
    .Add("Newtonsoft", () => NewtonsoftJson.Parse(data))
    .WithBaseline("System.Text.Json")
    .WithIterations(500)
    .WithWarmup(50)
    .WithAllocations()
    .WithOutlierMode(OutlierMode.IqrFence)
    .WithConfidenceLevel(0.99)
    .WithReporter(new MarkdownReporter("results.md"))
    .WithReporter(new CsvReporter("results.csv"))
    .RunAsync();
```

### Suite setup/teardown

```csharp
var results = await new BenchmarkSuite("File Processing")
    .Add("ReadAll", () => File.ReadAllBytes(path))
    .Add("Stream", () => StreamFile(path))
    .WithSuiteSetup(() => Directory.CreateTempSubdirectory())
    .WithSuiteTeardown(() => CleanupTempFiles())
    .RunAsync();
```

### Significant results

```csharp
foreach (var result in results)
{
    Console.WriteLine($"{result.Name}: median={result.Median:F1}ns, " +
        $"sig={result.SignificanceVerdict}, p-value={result.PValue}");
}
```

## MeasurementOptions

`MeasurementOptions` is a `record` with `init` properties. Use `with` expressions to override defaults.

### Defaults

| Property                     | Default             | Valid range |
| ---------------------------- | ------------------- | ----------- |
| `Iterations`                 | 200                 | 0 - 100,000 |
| `WarmupIterations`           | 25                  | 0 - 10,000  |
| `ForceGcBeforeEachIteration` | true                | -           |
| `MeasureAllocations`         | false               | -           |
| `OutlierMode`                | `RemoveTop5Percent` | -           |
| `ConfidenceLevel`            | 0.95                | >0 and <1   |
| `EnableSignificance`         | true                | -           |
| `ForceGcBetweenBenchmarks`   | true                | -           |

### OutlierMode values

| Value                        | Behaviour                                       |
| ---------------------------- | ----------------------------------------------- |
| `None`                       | Keep all samples (sort only)                    |
| `RemoveTop5Percent`          | Trim slowest 5% (default)                       |
| `RemoveTopAndBottom5Percent` | Trim 5% from each end                           |
| `IqrFence`                   | Remove samples outside Q1-1.5*IQR to Q3+1.5*IQR |

### Validating options

Property setters throw `ArgumentOutOfRangeException` for invalid values. Always check ranges.

### Dry run signal

`Iterations=0` and `WarmupIterations=0` together signal a dry run: the body is not invoked, and results are zeroed.

## Understanding BenchmarkResult fields

| Field                     | Description                                                       |
| ------------------------- | ----------------------------------------------------------------- |
| `Mean`                    | Arithmetic mean of measured samples (after outlier trim)          |
| `Median`                  | 50th percentile - the primary metric (more robust than mean)      |
| `P95`                     | 95th percentile (nearest-rank, ceil-based)                        |
| `P99`                     | 99th percentile                                                   |
| `Min`                     | Minimum sample                                                    |
| `Max`                     | Maximum sample                                                    |
| `StandardDeviation`       | Sample standard deviation (Bessel's correction, n-1)              |
| `StandardError`           | Standard error of the mean: StdDev / sqrt(n)                      |
| `MarginOfError`           | Half-width of the confidence interval on the mean                 |
| `ConfidenceLevel`         | The confidence level used (e.g. 0.95)                             |
| `ConfidenceIntervalLower` | Mean - MarginOfError                                              |
| `ConfidenceIntervalUpper` | Mean + MarginOfError                                              |
| `CoefficientOfVariation`  | StdDev / Mean (0 when mean is 0)                                  |
| `MeanAllocatedBytes`      | Average bytes allocated per iteration (if measured)               |
| `PValue`                  | Mann-Whitney U p-value against baseline (if significance enabled) |
| `SignificanceVerdict`     | `NotTested`, `Significant`, or `NotSignificant`                   |
| `Errored`                 | Whether the benchmark threw                                       |
| `ErrorMessage`            | Exception message if errored                                      |
| `MeasuredIterations`      | Number of iterations after outlier trim                           |
| `WarmupIterations`        | Number of warmup iterations                                       |
| `TotalDuration`           | End-to-end wall clock including warmup                            |
| `MeasuredDuration`        | Wall clock of measured loop only                                  |

## Key Concepts

### Warmup

Default 25 iterations that JIT and warm CPU caches. Timings are discarded.

### Garbage Collection

`ForceGcBeforeEachIteration` (default true) clears the GC heap before every warmup and measured iteration. `ForceGcBetweenBenchmarks` (default true) does a full gen-2 collect between benchmarks in a suite.

### Allocation Tracking

Enabled via `MeasureAllocations = true`. Uses thread-local GC stats with fallback to process-wide for async thread hops.

### Statistical Significance

When >=2 benchmarks and `EnableSignificance = true`, NBenchmark runs a two-tailed Mann-Whitney U test against the baseline (first explicit baseline, or the benchmark with the fastest median). Results show `✓` (significant) or `~` (not significant).

### Confidence Interval

The `MarginOfError` is `StudentT.CriticalValue(confidenceLevel, df) * StandardError`. The true mean lies within `Mean ± MarginOfError` at the given confidence level.

## Common Patterns

### Preventing dead-code elimination

Always return a value from the benchmarked code, or use `Benchmark.Run<T>(Func<T>)`. Internally, the runner uses a result-consuming sink to prevent the JIT from eliminating the computation.

```csharp
// GOOD: return value prevents elimination
Benchmark.Run(() => ComputeHash(data));

// GOOD: use Run<T> even if you don't need the value
Benchmark.Run(() => int.Parse("12345"));

// BAD: void method with no observable side effect
Benchmark.Run(() => { var x = 1 + 2; });
```

### Async benchmarks

Always use `RunAsync` / `RunAsync<T>` for async methods.

```csharp
await Benchmark.RunAsync(async () => await httpClient.GetStringAsync(url));
```

### Measuring very fast operations

Increase `Iterations` (e.g., 1000+) to get stable measurements for sub-microsecond operations. The Stopwatch typically has ~100ns resolution.

### Checking if a benchmark errored

```csharp
if (result.Errored)
    Console.WriteLine($"Failed: {result.ErrorMessage}");
```
