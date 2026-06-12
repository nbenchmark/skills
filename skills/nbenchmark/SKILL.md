---
name: nbenchmark
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-12
description: Core NBenchmark skill for writing .NET benchmarks with Quick mode (Benchmark.Run / RunAsync) and Suite mode (BenchmarkSuite fluent builder). Use when the user wants to benchmark a single method, compare multiple implementations, configure measurement options, interpret results, or prevent dead-code elimination. For dedicated benchmark projects with [Benchmark] attributes and a CLI, see nbenchmark-host. For output formats see nbenchmark-reporters; for failing tests on regressions see nbenchmark-integration.
---

# NBenchmark

NBenchmark is a lightweight, async-native, zero-dependency .NET benchmarking library (targets net8.0/net9.0/net10.0). It has three usage modes that share one measurement engine and produce the same `BenchmarkResult` type:

| Mode      | Entry point                  | Use for                                       | Skill             |
| --------- | ---------------------------- | --------------------------------------------- | ----------------- |
| **Quick** | `Benchmark.Run` / `RunAsync` | A single one-off measurement                  | this skill        |
| **Suite** | `BenchmarkSuite` (fluent)    | Comparing 2+ implementations A/B              | this skill        |
| **Host**  | `BenchmarkHost`              | Dedicated benchmark projects, attributes, CLI | `nbenchmark-host` |

Install: `dotnet add package NBenchmark`. Add `NBenchmark.Reporters.Console` for rich terminal output.

> The rich console package is `NBenchmark.Reporters.Console` (namespace `NBenchmark.Reporters.Console`). File reporters (JSON/Markdown/CSV) are built into the core `NBenchmark` package.

## When to use this skill

- Add benchmarking to a .NET project
- Benchmark a single method (`Benchmark.Run`)
- Compare two or more implementations (`BenchmarkSuite`)
- Configure iterations, warmup, outlier mode, confidence level, or significance level
- Understand what the output numbers mean
- Prevent dead-code elimination in benchmarks

## Quick Mode

Eight static methods on `Benchmark`. Every overload takes `(delegate, MeasurementOptions? options = null, string name = "Benchmark", CancellationToken cancellationToken = default)`.

```csharp
using NBenchmark;

// Sync void
BenchmarkResult r1 = Benchmark.Run(() => Array.Sort(data));

// Sync returning — the result is consumed by a sink to prevent dead-code elimination
BenchmarkResult r2 = Benchmark.Run(() => ComputeHash(data));

// Async void / returning
BenchmarkResult r3 = await Benchmark.RunAsync(async () => await FetchDataAsync());
BenchmarkResult r4 = await Benchmark.RunAsync(async () => await LoadFromDbAsync());

r1.Print();   // plain-text summary (core package)
```

### Raw samples (`RunRaw*`)

The four `RunRaw` / `RunRaw<T>` / `RunRawAsync` / `RunRawAsync<T>` methods return a `MeasurementOutcome` exposing the pre-trim timing array:

```csharp
MeasurementOutcome outcome = Benchmark.RunRaw(() => ComputeHash(data));
double[] rawTimings = outcome.RawSamples;   // nanoseconds, before outlier trimming
BenchmarkResult result = outcome.Result;
```

### Options and naming

```csharp
var result = Benchmark.Run(() => MyMethod(),
    options: new MeasurementOptions
    {
        Iterations = 500,
        WarmupIterations = 50,
        ConfidenceLevel = 0.99,
        MeasureAllocations = true,
        OutlierMode = OutlierMode.IqrFence,
    },
    name: "MyMethod-v2");
```

### Single-result file output (extension methods)

```csharp
await result.ToMarkdownAsync("results/");          // optional 2nd arg: fileName
await result.ToJsonAsync("results/");
await result.ToCsvAsync("results.csv");
await result.PrintAsync();   // rich console table — requires NBenchmark.Reporters.Console
```

## Suite Mode

`BenchmarkSuite` compares implementations side-by-side with ratios and statistical significance against a baseline.

```csharp
using NBenchmark;
using NBenchmark.Reporters.Console;

var results = await new BenchmarkSuite("Sorting")
    .Add("BubbleSort", () => BubbleSort(data))
    .Add("QuickSort",  () => QuickSort(data))
    .WithBaseline("QuickSort")
    .WithReporter(new ConsoleReporter())
    .RunAsync();
```

### Four `Add` overloads

Each takes `(string name, delegate, Action? setup = null, Action? teardown = null)`:

```csharp
suite.Add("a", () => DoSomething());                       // sync void
suite.Add("b", () => ComputeSomething());                  // sync returning (prevents DCE)
suite.Add("c", async () => await DoSomethingAsync());      // async void
suite.Add("d", async () => await ComputeSomethingAsync()); // async returning
```

- `setup` / `teardown` run before/after **each iteration** and are **not** included in the measurement.
- Benchmark names must be **unique** within a suite (significance keys raw samples by name; duplicates throw `ArgumentException`).

### Fluent configuration

| Method                                                 | What it sets                                          |
| ------------------------------------------------------ | ----------------------------------------------------- |
| `WithBaseline(name)`                                   | Reference benchmark for ratio/significance            |
| `WithIterations(n)`                                    | Measured iterations (default 200)                     |
| `WithWarmup(n)`                                        | Warmup iterations (default 25)                        |
| `WithAllocations(bool = true)`                         | Per-iteration allocation tracking                     |
| `WithOutlierMode(mode)`                                | Outlier trimming strategy (default `IqrFence`)        |
| `WithConfidenceLevel(level)`                           | Confidence level for the Error column (default 0.95)  |
| `WithSignificance(bool)`                               | Enable/disable Mann-Whitney U test (default enabled)  |
| `WithSignificanceLevel(level)`                         | Alpha for the significance test (default 0.05)        |
| `WithRunOrder(order)`                                  | `RunOrder.Random` (default) or `RunOrder.Declaration` |
| `WithSuiteSetup(action)` / `WithSuiteTeardown(action)` | Run once around the whole suite                       |
| `WithReporter(reporter)`                               | Add an `IReporter` (stackable)                        |
| `WithDetail(detail)`                                   | `ReportDetail.Simple` (default) or `.Advanced`        |
| `WithProgress(progress)`                               | Live progress callback (`IBenchmarkProgress`)         |

`RunAsync()` returns `IReadOnlyList<BenchmarkResult>`. Errored benchmarks are included with `Errored == true` and `ErrorMessage` set. Suite teardown is guaranteed to run once setup succeeds, even on cancellation.

```csharp
var results = await new BenchmarkSuite("JSON Parsers")
    .Add("System.Text.Json", () => SystemTextJson.Parse(data))
    .Add("Newtonsoft",       () => NewtonsoftJson.Parse(data))
    .WithBaseline("System.Text.Json")
    .WithIterations(500).WithWarmup(50)
    .WithAllocations()
    .WithOutlierMode(OutlierMode.IqrFence)
    .WithConfidenceLevel(0.99)
    .WithSignificanceLevel(0.01)
    .WithReporter(new MarkdownReporter("results/"))
    .RunAsync();
```

## MeasurementOptions defaults

`MeasurementOptions` is a `record` with `init` properties. Setters throw `ArgumentOutOfRangeException` for out-of-range values.

| Property                     | Default                | Valid range |
| ---------------------------- | ---------------------- | ----------- |
| `Iterations`                 | `200`                  | 0–100,000   |
| `WarmupIterations`           | `25`                   | 0–10,000    |
| `ForceGcBeforeEachIteration` | `true`                 | —           |
| `MeasureAllocations`         | `false`                | —           |
| `OutlierMode`                | `OutlierMode.IqrFence` | —           |
| `ConfidenceLevel`            | `0.95`                 | >0 and <1   |
| `EnableSignificance`         | `true`                 | —           |
| `SignificanceLevel`          | `0.05`                 | >0 and <1   |
| `ForceGcBetweenBenchmarks`   | `true`                 | —           |

`Iterations = 0` **and** `WarmupIterations = 0` together signal a dry run: the body is not invoked and results are zeroed.

See [references/measurement-options.md](references/measurement-options.md) for a full explanation of every option, outlier modes, and tuning advice.

## Interpreting results

The primary metric is the **Median** (robust to outliers). Key fields:

- `Median`, `Mean`, `P95`, `P99`, `Min`, `Max`
- `StandardDeviation`, `StandardError`, `MarginOfError` (the "Error" column), `ConfidenceLevel`
- `MeanAllocatedBytes` (when allocations measured)
- `PValue`, `SignificanceVerdict` (`NotTested` / `Significant` / `NotSignificant`)
- `Errored`, `ErrorMessage`
- `Warnings` — non-fatal notes such as a bimodal-distribution warning

`BenchmarkResult` carries the full distribution (`Q1`, `Q3`, `InterquartileRange`, `LowerFence`, `UpperFence`, `Skewness`, `Kurtosis`, `Mad`, allocation percentiles, etc.). See [references/benchmark-result.md](references/benchmark-result.md) for the complete field list.

### Statistical significance

With ≥2 non-errored benchmarks and `EnableSignificance = true`, NBenchmark runs a two-sided **Mann-Whitney U test** on the pre-trim raw samples against the baseline. The baseline is the first explicit baseline, or the benchmark with the fastest median. A result is **Significant** when its p-value < `SignificanceLevel` (default 0.05).

- The test needs at least **2 samples per group**; with fewer it returns no result (`NotTested`).
- For small, tie-free samples (combined n ≤ 20) an exact permutation p-value is used; otherwise the normal approximation with tie and continuity corrections.
- Sig column: `✓` significant, `✗` not significant, `-` not applicable (baseline / disabled / not tested).
- Significance ≠ importance — always read the **Ratio** column alongside it.

## Common patterns

### Prevent dead-code elimination

Return a value from the benchmarked code, or use a value-returning overload. The runner consumes returned values through a sink so the JIT can't elide the work.

```csharp
Benchmark.Run(() => ComputeHash(data));   // GOOD: return value is consumed
Benchmark.Run(() => int.Parse("12345"));  // GOOD
Benchmark.Run(() => { var x = 1 + 2; });  // BAD: no observable side effect → may report 0 ns
```

### Async

Always use `RunAsync` / `RunAsync<T>` (or the `Func<Task>` `Add` overloads) for async work. The timer captures the full awaited duration.

### Very fast operations

Increase `Iterations` (1000+) for stable sub-microsecond measurements. Per-iteration timing preserves native sub-100 ns stopwatch resolution.

### Detecting errors

```csharp
if (result.Errored)
    Console.WriteLine($"Failed: {result.ErrorMessage}");
```

## Related skills

- **nbenchmark-host** — attribute-based discovery, CLI, dependency injection, `[IsolatedProcess]`, CI regression gates
- **nbenchmark-reporters** — console/JSON/Markdown/CSV output, detail levels, custom reporters
- **nbenchmark-integration** — enforce performance thresholds as xUnit/NUnit/MSTest tests
- **nbenchmark-troubleshooting** — analyzer diagnostics (NB0001–NB0010), wrong results, tuning
