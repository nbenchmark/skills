---
name: nbenchmark
nbenchmarkVersion: v0.37.0
lastVerified: 2026-08-01
description: Core NBenchmark skill for writing .NET benchmarks with Single mode (Benchmark.Run / RunAsync) and Suite mode (BenchmarkSuite fluent builder). Use when the user wants to benchmark a single method, compare multiple implementations, configure measurement options, interpret results, or prevent dead-code elimination. Covers worker-process isolation (the default in every mode), prepared state, and runtime profiles. For dedicated benchmark projects with [Benchmark] attributes and a CLI, see nbenchmark-host. For output formats see nbenchmark-reporters; for failing tests on regressions see nbenchmark-integration.
---

# NBenchmark

NBenchmark is a lightweight, async-native, zero-dependency .NET benchmarking library (targets net8.0/net9.0/net10.0). It has three usage modes that share one measurement engine and produce the same `BenchmarkResult` type:

| Mode       | Entry point                  | Use for                                       | Skill             |
| ---------- | ---------------------------- | --------------------------------------------- | ----------------- |
| **Single** | `Benchmark.Run` / `RunAsync` | A single one-off measurement                  | this skill        |
| **Suite**  | `BenchmarkSuite` (fluent)    | Comparing 2+ implementations A/B              | this skill        |
| **Harness**| `BenchmarkHarness`           | Dedicated benchmark projects, attributes, CLI | `nbenchmark-host` |

Install: `dotnet add package NBenchmark`. Add `NBenchmark.Reporters.Console` for rich terminal output.

> The rich console package is `NBenchmark.Reporters.Console` (namespace `NBenchmark.Reporters.Console`). File reporters (JSON/Markdown/CSV) are built into the core `NBenchmark` package.

## Process isolation

Every mode - Single, Suite, Harness, and the test-framework integrations - measures in a **dedicated worker process by default**. JIT tiering, dynamic PGO, and GC flavour are fixed at process start, so a benchmark measured in a host process inherits whatever the preceding benchmarks left behind. The worker is a dedicated `nbworker` executable launched with a known runtime configuration.

A benchmark body that captures a local variable, parameter, or `this` from its enclosing scope cannot be addressed in a worker and falls back to the host with a labelled reason (`IsolationStatus`). In Suite mode, one capturing body takes every sibling in-process. The remedy is **prepared state** - pass the preparation as its own delegate so the worker builds the state itself.

```csharp
var data = BuildData();   // a local in the caller's scope

// Captures `data` -> runs in-process (InProcessCapturedState)
Benchmark.Run(() => Sort(data));

// Prepared state -> runs in a worker
Benchmark.Run(prepare: () => BuildData(), body: d => Sort(d));
```

Deliberate host measurement: `Benchmark.RunInProcess`. Opt a suite out: `WithIsolation(false)`. The result's `IsolationStatus` field names exactly why a measurement ran where it did, and an `Iso` column appears in mixed-isolation tables. See [references/isolation.md](references/isolation.md) for the full worker model, runtime profiles, and the capture fallback.

## When to use this skill

- Add benchmarking to a .NET project
- Benchmark a single method (`Benchmark.Run`)
- Compare two or more implementations (`BenchmarkSuite`)
- Configure iterations, warmup, outlier mode, confidence level, or significance level
- Understand what the output numbers mean
- Prevent dead-code elimination in benchmarks
- Run a benchmark in a worker process (the default) or deliberately in-process
- Pass prepared state so a worker can build it (the capture-fallback remedy)

## Single Mode

The `Benchmark` static class exposes three families: `Run` / `RunAsync` (measure in a worker by default), `RunInProcess` / `RunInProcessAsync` (measure in this process, deliberately), and `RunRaw*` (return the raw-sample array). Every overload takes `(delegate, MeasurementOptions? options = null, string name = "Benchmark", CancellationToken cancellationToken = default)`.

```csharp
using NBenchmark;

// Sync void
BenchmarkResult r1 = Benchmark.Run(() => Array.Sort(data));

// Sync returning — the result is consumed by a sink to prevent dead-code elimination
BenchmarkResult r2 = Benchmark.Run(() => ComputeHash(data));

// Async void / returning
BenchmarkResult r3 = await Benchmark.RunAsync(async () => await FetchDataAsync());
BenchmarkResult r4 = await Benchmark.RunAsync(async () => await LoadFromDbAsync());

r1.Print();   // plain-text summary (core package; optional detail arg defaults to Simple)
```

`Run` / `RunAsync` measure in a worker by default. A body that captures local state falls back to the host (`IsolationStatus.InProcessCapturedState`); use the prepared-state overloads below, or `Benchmark.RunInProcess` when host measurement is the intent.

### Prepared state (`Run` with `prepare`)

Pass the preparation as its own delegate so the worker builds the state in the process that measures it. The remedy for the capture fallback.

```csharp
// prepare runs once per benchmark, before warmup, in the worker; body receives what it returned
BenchmarkResult r = Benchmark.Run(
    prepare: () => BuildData(),
    body:    d  => Sort(d));
```

Same shape for sync/async, void/returning, and `RunRaw` variants - `Func<TState> prepare` is prepended to the signature. Both `prepare` and `body` must capture nothing. `prepare` runs once, not per iteration - a body that mutates its state sees the mutation on every iteration after the first.

### Deliberate in-process (`RunInProcess`)

`Benchmark.RunInProcess` / `RunInProcessAsync` measure in this process deliberately and without a warning - the right choice, not a fallback, for cold-start/first-call cost, a body that must observe host state, or a comparison against a number produced before workers existed. Stamps `IsolationStatus.InProcessRequested`. Same overload shapes as `Run`.

### Pre-warming a worker (`Warmup`)

```csharp
Benchmark.Warmup();   // pre-warm a worker in the background
```

Optional. A worker costs roughly 70 ms to start; `Benchmark.Warmup()` hides that cost from the first `Benchmark.Run`. Worth calling at the start of a script, or before a timed section of a tool. Fire-and-forget by design - the run that needs the worker will report any failure better than this can. See [references/isolation.md](references/isolation.md#benchmarkwarmup).

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
        OutlierMode = OutlierMode.IqrFence,
    },
    name: "MyMethod-v2");
```

Note: allocation tracking defaults to `true`. Use `MeasureAllocationsOverride = false` to disable.

### Single-result file output (extension methods)

```csharp
await result.ToMarkdownAsync("results/");          // optional 2nd arg: fileName
await result.ToJsonAsync("results/");
await result.ToCsvAsync("results/");
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

### `Add` overloads

`Add` registers a benchmark. The simplest form takes `(string name, delegate, Action? setup = null, Action? teardown = null, IReadOnlyList<string>? categories = null)`:

```csharp
suite.Add("a", () => DoSomething());                       // sync void
suite.Add("b", () => ComputeSomething());                  // sync returning (prevents DCE)
suite.Add("c", async () => await DoSomethingAsync());      // async void
suite.Add("d", async () => await ComputeSomethingAsync()); // async returning
```

- `setup` / `teardown` run before/after **each iteration** and are **not** included in the measurement.
- Benchmark names must be **unique** within a suite (significance keys raw samples by name; duplicates throw `ArgumentException`).
- `categories` tags the entry for `WithCategoryFilter` (Harness mode) and shows up in the `Categories` column.

### Parameterised `Add` (typed lambdas)

For suite-mode parameterised benchmarks, use the `Add<T>` / `Add<T1, T2>` / `Add<T1, T2, T3>` overloads that take a delegate with parameters, then register parameter values with `WithParameter<T>(...)`. The runner expands the cartesian product of registered parameters, producing one result row per combination:

```csharp
await new BenchmarkSuite("Sorting")
    .Add<int>("Sort", n => Array.Sort(Enumerable.Range(0, n).Reverse().ToArray()))
    .WithParameter("size", 10, 100, 1_000, 10_000)
    .WithBaseline("Sort(size=10)")   // name auto-derived from parameter values
    .RunAsync();
```

Arity-2 and arity-3 overloads take `Func<T1, T2, TResult>` / `Func<T1, T2, T3, TResult>` (and async equivalents). Pair with `WithParameter<T1, T2>(name1, values1, name2, values2)` and `WithParameter<T1, T2, T3>(...)`.

### Fluent configuration

| Method                                                 | What it sets                                          |
| ------------------------------------------------------ | ----------------------------------------------------- |
| `WithBaseline(name)`                                   | Reference benchmark for ratio/significance            |
| `WithLaunchCount(n)`                                   | Repeat the suite as `n` separate launches (default 1) |
| `WithIterations(n)`                                    | Pin measured samples (default: auto)                  |
| `WithWarmup(n)`                                        | Pin warmup samples (default: auto)                    |
| `WithOpsPerSample(n)`                                  | Pin ops-per-sample / K (default: auto-calibrated)     |
| `WithAutoTune(preset)` / `WithAutoTune(options)`       | Bound/steer the adaptive loop                         |
| `WithAllocations(bool = true)`                         | Per-iteration allocation tracking                     |
| `WithDiagnostics(options)` / `WithDiagnostics(mode)`   | GC/heap/exception/CPU-time diagnostics                |
| `WithMeasurementProfile(profile)`                      | `Realistic` (default) or `Independent` GC behaviour   |
| `WithOutlierMode(mode)`                                | Outlier trimming strategy (default `IqrFence`)        |
| `WithOutlierDetector(detector)`                        | Custom `IOutlierDetector` (overrides mode)            |
| `WithConfidenceLevel(level)`                           | Confidence level for the Error column (default 0.95) |
| `WithSignificance(bool)`                               | Enable/disable significance test (default enabled)   |
| `WithSignificanceLevel(level)`                         | Alpha for the significance test (default 0.05)        |
| `WithSignificanceTest(test)`                            | Custom `ISignificanceTest` strategy                   |
| `WithMinimumPracticalEffect(delta)`                    | Downgrade Sig to NotSignificant below this effect size |
| `WithHardwareAffinity(params int[] cores)`             | Pin CPU cores for the run                              |
| `WithProcessPriority(priority)`                        | Set process priority                                   |
| `WithDedicatedHostGuidance(bool = true)`               | Warn if running on a shared CI host                   |
| `WithRunOrder(order)`                                  | `RunOrder.Random` (default) or `RunOrder.Declaration` |
| `WithSuiteSetup(action)` / `WithSuiteTeardown(action)` | Run once around the whole suite                       |
| `WithReporter(reporter)`                               | Add an `IReporter` (stackable)                        |
| `WithDetail(detail)`                                   | `ReportDetail.Simple` (default) / `Standard` / `Advanced` |
| `WithProgress(progress)`                               | Live progress callback (`IBenchmarkProgress`)         |
| `WithObserver(observer)`                               | Non-perturbing `IMeasurementObserver` telemetry       |
| `WithCategories(params string[])`                      | Tag every entry in the suite                          |
| `WithCategoryFilter(include?, exclude?)`               | Run only matching categories                          |
| `WithIsolation(bool = true)`                           | Superseded - isolation is the default. `WithIsolation(false)` opts back into the host, deliberately and silently |
| `WithRuntimeProfile(profile)`                          | Runtime-startup config (`SteadyState`/`Production`/`ServerGc`/`Host`); default `SteadyState` |
| `WithRequireIsolation(bool = true)`                    | Fail the run if any body falls back to in-process (non-opt-out) |
| `WithState(prepare)`                                   | Prepared state - the worker builds it. Returns `BenchmarkSuite<TState>` |
| `WithRuntimes(params RuntimeMoniker[])`                | Run across .NET runtimes (`Net8`/`Net9`/`Net10`); requires `[BenchmarkPlan]` |

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

### Suite isolation

A suite measures in a **single worker process** by default, so all ratios are paired within-process comparisons - no `WithIsolation()` needed. One capturing `Add` body degrades the whole suite to in-process, because a paired ratio must be a within-process statement. Two escape hatches when a suite holds live state:

```csharp
// Prepared state - the worker builds it
var results = await new BenchmarkSuite("Sorting")
    .WithState(() => BuildData())
    .Add("BubbleSort", d => BubbleSort(d))
    .Add("QuickSort",  d => QuickSort(d))
    .WithBaseline("QuickSort")
    .RunAsync();

// [BenchmarkPlan] - the worker invokes the factory itself; nothing needs to be serializable
await BenchmarkSuite.RunPlanAsync(BuildSuite);

static BenchmarkSuite BuildSuite() =>
    new BenchmarkSuite("serialization")
        .Add("json", () => SerializeJson())
        .Add("msgpack", () => SerializeMsgPack())
        .WithBaseline("json");
```

`WithRequireIsolation()` fails the run if any body falls back to in-process for a non-opt-out reason. Multi-runtime suite runs (`WithRuntimes`) require `[BenchmarkPlan]`, because benchmark bodies are addressed by metadata token which is only valid within the build that produced it. A suite carrying both `WithState` and `WithParameter` cannot isolate - the parameter sweep and the state factory are not composable across a process boundary. See [references/isolation.md](references/isolation.md) for the full story.

## MeasurementOptions defaults

`MeasurementOptions` is a `record` with `init` properties. Setters throw `ArgumentOutOfRangeException` for out-of-range values.

| Property                     | Default                | Valid range |
| ---------------------------- | ---------------------- | ----------- |
| `Iterations`                 | `null` (auto)          | 0–100,000    |
| `WarmupIterations`           | `null` (auto)          | 0–10,000     |
| `OpsPerSample`               | `null` (auto)          | 1–16,777,216 |
| `AutoTune`                   | `AutoTuneOptions.Default` | —         |
| `Diagnostics`                | `DiagnosticsOptions.Default` (GC counts on) | — |
| `Profile`                    | `MeasurementProfile.Realistic` | `Realistic` / `Independent` |
| `ForceGcBeforeEachIteration` | derives from `Profile` (true under `Independent`, false under `Realistic`) | — |
| `ForceGcBeforeMeasurement`   | derives from `Profile` | —           |
| `ForceGcBetweenBenchmarks`   | `true`                 | —           |
| `MeasureAllocations`         | `true`                 | —           |
| `OutlierMode`                | `OutlierMode.IqrFence` | `None` / `RemoveTop5Percent` / `RemoveTopAndBottom5Percent` / `IqrFence` / `MedianAbsoluteDeviation` |
| `OutlierDetector`            | `null` (uses `OutlierMode`) | any `IOutlierDetector` |
| `TailMetricsBasis`           | `TailMetricsBasis.Raw` | `Raw` / `Trimmed` |
| `ConfidenceLevel`            | `0.95`                 | >0 and <1   |
| `ReportedPercentiles`        | `[0.50, 0.95, 0.99, 0.999, 1.0]` | each in [0, 1] |
| `EnableHistogram`            | `true`                 | —           |
| `HistogramBucketCount`       | `20`                   | 5–100       |
| `EnableSignificance`         | `true`                 | —           |
| `SignificanceTest`           | `null` (uses `DefaultSignificanceTest`) | any `ISignificanceTest` |
| `SignificanceLevel`          | `0.05`                 | >0 and <1   |
| `MinimumPracticalEffect`     | `0.147`                | [0, 1] or `null` to disable |
| `MaxRawSamples`              | `4096` (`DefaultMaxRawSamples`) | `0` (`UnboundedRawSamples`) or positive |
| `StreamSamples`              | `false`                | —           |
| `SuppressPerClassIndependenceWarning` | `false`       | —           |
| `Environment`                | `null`                 | `EnvironmentOptions` (CPU affinity / priority / dedicated-host guidance) |

`LaunchCount` is **not** a `MeasurementOptions` property - it lives on the `LaunchCounts` static class and is passed by the coordinator that spends it (`WithLaunchCount(n)`, `--launch-count`, `[Benchmark(LaunchCount = n)]`). See [references/measurement-options.md](references/measurement-options.md#launchcount).

`Iterations = 0` signals a dry run: the measured loop is skipped and results are zeroed (the body isn't invoked unless a positive `WarmupIterations` is pinned). `--dry-run` sets both counts to `0`.

See [references/measurement-options.md](references/measurement-options.md) for a full explanation of every option, outlier modes, and tuning advice.

## Interpreting results

The primary metric is the **Median** (robust to outliers). Key fields:

- `Median`, `Mean`, `Min`, `Max`
- `Percentiles` — `IReadOnlyList<PercentileEntry>` with the configured percentiles (default P50/P95/P99/P99.9/Max). Use `result.GetPercentile(0.95)` for convenience.
- `Histogram` — `LatencyHistogram?` of trimmed samples (buckets + counts), `null` when disabled or <2 samples
- `StandardDeviation`, `StandardError`, `MarginOfError` (the "Error" column), `ConfidenceLevel`
- `MedianCiLower` / `MedianCiUpper` — distribution-free confidence interval on the median
- `MedianShift` — Hodges-Lehmann shift vs baseline with rank-based CI (`null` on baseline / single runs)
- `OperationsPerSecond`, `MedianOperationsPerSecond` — `1e9 / Mean` and `1e9 / Median`
- `AllocMedian`, `AllocP95`, `AllocMax`, `MeanAllocatedBytes` (when allocations measured)
- `PValue`, `SignificanceVerdict` (`NotTested` / `Significant` / `NotSignificant`), `Effect` (Cliff's δ effect size), `Omnibus` (Kruskal-Wallis verdict for 3+ groups)
- `SignificanceTestName`, `SignificanceLevel`
- `IsolationStatus` — where the measurement ran and why (`Isolated` / `InProcessRequested` / `InProcessCapturedState` / `InProcessLiveFixture` / `InProcessUnaddressablePlan` / `InProcessNoWorker`). Shown as the `Iso` column in mixed-isolation tables.
- `RuntimeProfileName`, `RuntimeKnobs` — the runtime-startup configuration actually measured under. Results under different profiles are never compared.
- `LaunchStatistics` — populated when the launch count > 1 (per-launch medians, mean, stddev, CI, plus the reproducibility diagnostics `ProcessVarianceRatio` / `BetweenLaunchDispersion`)
- `Diagnostics` — `DiagnosticsResult?` with GC counts, heap info, exceptions/sec, CPU time when enabled
- `Categories`, `ParameterSet`, `RuntimeMoniker`
- `AutoTune` — `AutoTuneDiagnostic?` of what the adaptive loop resolved
- `Errored`, `ErrorMessage`
- `Warnings` — non-fatal notes such as a bimodal-distribution warning

`BenchmarkResult` carries the full distribution (`Q1`, `Q3`, `InterquartileRange`, `LowerFence`, `UpperFence`, `Skewness`, `Kurtosis`, `Mad`, `TrimmedOrdinals`, `N`, `OutliersRemoved`, etc.). See [references/benchmark-result.md](references/benchmark-result.md) for the complete field list.

### Statistical significance

With ≥2 non-errored benchmarks and `EnableSignificance = true`, NBenchmark runs a significance test on the pre-trim raw samples against the baseline. The default strategy is **Mann-Whitney U** for two groups and **Kruskal-Wallis** with post-hoc pairwise tests for three or more. The baseline is the first explicit baseline, or the benchmark with the fastest median. A result is **Significant** when its p-value < `SignificanceLevel` (default 0.05).

- The test needs at least **2 samples per group**; with fewer it returns no result (`NotTested`).
- For small, tie-free samples (combined n ≤ 20) an exact permutation p-value is used; otherwise the normal approximation with tie and continuity corrections.
- The `MinimumPracticalEffect` gate (default 0.147 - a "small" Cliff's δ) downgrades a statistically-significant but practically-negligible result to `NotSignificant`. Set to `0` for p-value-only semantics, or `null` to disable the gate.
- Sig column: `✓` significant, `✗` not significant, `-` not applicable (baseline / disabled / not tested).
- Significance ≠ importance — always read the **Ratio** column alongside it.
- A ratio is only formed between rows measured under the same runtime configuration. Mixed-isolation tables show `n/a` for the ratio, an `Iso` column identifying where each row ran, and a footer explaining the withholding. A mixed-process ratio would describe two processes more than two code paths.
- With two or more launches, the ratio is a **paired** per-replicate estimate (geometric mean with a multiplicative CI); an interval spanning 1.0 means the two benchmarks are not distinguishable at that replicate count. See [references/significance-and-outliers.md](references/significance-and-outliers.md#paired-ratio-estimation).

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

For fast, side-effect-free bodies the loop auto-calibrates ops-per-sample (K) so each timed sample spans enough work to beat timer resolution, dividing the per-op time back down. Pin it with `WithOpsPerSample(n)` (or `--ops-per-sample n`), or pin a larger `Iterations` budget, when you want a fixed sub-microsecond measurement.

### Detecting errors

```csharp
if (result.Errored)
    Console.WriteLine($"Failed: {result.ErrorMessage}");
```

## References

- [isolation.md](references/isolation.md) - worker model, capture fallback, prepared state, `[BenchmarkPlan]`, runtime profiles, `IsolationStatus`
- [benchmark-result.md](references/benchmark-result.md) - every `BenchmarkResult` field
- [measurement-options.md](references/measurement-options.md) - every `MeasurementOptions` property, outlier modes, tuning
- [significance-and-outliers.md](references/significance-and-outliers.md) - pluggable significance tests, outlier detectors, effect size, paired ratio estimation, warnings

## Related skills

- **nbenchmark-host** — attribute-based discovery, CLI, dependency injection, `[IsolatedProcess]`/`[InProcess]`, CI regression gates
- **nbenchmark-reporters** — console/JSON/Markdown/CSV output, detail levels, custom reporters
- **nbenchmark-integration** — enforce performance thresholds as xUnit/NUnit/MSTest tests
- **nbenchmark-troubleshooting** — analyzer diagnostics (NB0001-NB0014), wrong results, tuning
