---
name: nbenchmark-integration
nbenchmarkVersion: v0.37.0
lastVerified: 2026-08-01
description: NBenchmark test-framework integration. Use when the user wants to enforce performance thresholds (max mean, max P95, max allocations, baseline regression) as part of an existing xUnit, NUnit, or MSTest test suite — via [PerformanceFact]/[PerformanceTheory], [Performance], [PerformanceTestMethod], or PerformanceAssert.Run. Tests run in a worker process by default; RequireIsolation (default true) and [AllowInProcessGate] control that. For dedicated benchmark projects with a CLI regression gate (--threshold-pct) see nbenchmark-host instead.
---

# NBenchmark Test Integration

These packages let you assert performance budgets inside your **existing** test suite — no separate benchmark project or CI step. A test fails when the measured benchmark violates a threshold.

| Package                         | Framework      |
| ------------------------------- | -------------- |
| `NBenchmark.Integration.xUnit`  | xUnit v2       |
| `NBenchmark.Integration.NUnit`  | NUnit 3 / 4    |
| `NBenchmark.Integration.MSTest` | MSTest v2 / v3 |

All three depend on `NBenchmark` (core) and pull in the shared `NBenchmark.Integration.Abstractions` package automatically.

## Where your test is measured

A performance test is measured in a **worker process by default** - for the same reason the benchmark runner always has been: JIT tiering, dynamic PGO, and GC flavour are fixed at process start, and a test host's are whatever preceding tests left behind. A `TestBodyIsolation` classifier decides whether a test class can be reconstructed in a worker:

| Situation | Isolation status |
|---|---|
| Plain class with a parameterless constructor, simple-value arguments | `Isolated` |
| Static method | `Isolated` |
| `IClassFixture` / `ITestOutputHelper` / constructor injection | `InProcessLiveFixture` (cannot reproduce the fixture in a worker) |
| Object-graph arguments (anything but simple values) | `InProcessLiveFixture` |
| No worker deployed | `InProcessNoWorker` (deployment problem) |

Simple values (`int`, `string`, `decimal`, `DateTime`, `Guid`, etc.) reach the worker intact, so `[InlineData]` / `[DataRow]` / `[TestCase]` isolate normally. `RequireIsolation` (default `true`) fails the test when it was measured in the host; the opt-out is `[AllowInProcessGate]`.

> [!NOTE]
> A ratio gate is only enforced when both sides were measured the same way. A mixed-process ratio (worker vs test host) is skipped and logged, because the ratio would describe two processes more than two code paths. See the core skill's [isolation reference](../nbenchmark/references/isolation.md) for the shared worker model, `IsolationStatus`, and runtime profiles.

## When to use this skill

- Fail a unit test if code is slower than a budget
- Enforce a max allocation budget in tests
- Guard against regressions versus a reference method (or the built-in calibration baseline)
- Measure just one part of a larger test (`PerformanceAssert.Run`)
- Run a test in a worker process (the default) or waive the isolation requirement (`[AllowInProcessGate]`)
- Pair candidate and reference across launches for a reproducible ratio interval (`LaunchCount`)

> For a standalone benchmark project gated by `--threshold-pct` in CI, use the `nbenchmark-host` skill. This skill is for assertions embedded in xUnit/NUnit/MSTest tests.

## Two usage patterns

### 1. Attribute pattern (all three frameworks)

Replace the test attribute. The **entire method body** becomes the benchmark; thresholds are named arguments.

```csharp
// xUnit
[PerformanceFact(MaxMeanNs = 500_000)]
public void ParseJson() => JsonSerializer.Deserialize<MyDto>(Payload);

// NUnit
[Performance(MaxMeanNs = 500_000)]
public void ParseJson() => JsonSerializer.Deserialize<MyDto>(Payload);

// MSTest
[PerformanceTestMethod(MaxMeanNs = 500_000)]
public void ParseJson() => JsonSerializer.Deserialize<MyDto>(Payload);
```

If the mean exceeds 500,000 ns the test fails with a message describing the violation plus formatted metrics. xUnit also has `[PerformanceTheory]` (parameterized, like `[Theory]`).

### 2. Assert pattern (NUnit and MSTest only)

Call `PerformanceAssert.Run` inside any test to benchmark one part of it inline:

```csharp
[Test] // NUnit  (MSTest: [TestMethod])
public void Repository_Query_Is_Fast_Enough()
{
    var repo = new OrderRepository(connection);

    PerformanceAssert.Run(
        () => repo.GetRecentOrders(limit: 100),
        new PerformanceAssertionOptions { MaxMeanNs = 2_000_000 });
}
```

xUnit has **no** `PerformanceAssert`; for inline assertions in xUnit, run a benchmark with `Benchmark.Run(...)` and validate it with `BenchmarkAssert.Validate(...)` (see below).

## Thresholds

The attributes implement `IPerformanceThresholds` and expose **thirteen** properties. A `-1` (for `double`/`long`) or `0` (for `MaxSlowdownRatio` / `Iterations` / `WarmupIterations` / `LaunchCount`) means "disabled / use default"; omitting a property is the same as the default.

`RequireIsolation` is settable only on `PerformanceAssertionOptions` (assert pattern), not on the attributes - a named-attribute `bool` could not express `false` under xUnit, so the opt-out is `[AllowInProcessGate]`. See that section below.

| Property                        | Type          | Default               | Description                                               |
| ------------------------------- | ------------- | --------------------- | --------------------------------------------------------- |
| `MaxMeanNs`                     | `double`      | `-1` (disabled)       | Max allowed mean (ns)                                     |
| `MaxP95Ns`                      | `double`      | `-1` (disabled)       | Max allowed P95 (ns)                                      |
| `MaxAllocatedBytes`             | `long`        | `-1` (disabled)       | Max mean bytes/op; implicitly enables allocation tracking |
| `ReferenceMethod`               | `string?`     | `null`                | Reference method name to compare against (attribute pattern only) |
| `MaxSlowdownRatio`              | `double`      | `0` (disabled)        | Max slowdown vs reference (1.1 = +10%)                     |
| `Iterations`                    | `int`         | `0` (use default: auto) | Measured-sample override (`>0` pins)                    |
| `WarmupIterations`              | `int`         | `0` (use default: auto) | Warmup override (`>0` pins)                             |
| `LaunchCount`                   | `int`         | `0` (use default: 1)  | Worker processes per test; `>=2` turns on the paired ratio gate |
| `MeasureAllocations`            | `bool`        | `false`               | Enable allocation tracking                                |
| `OutlierMode`                   | `OutlierMode` | `IqrFence`            | Outlier strategy                                          |
| `ConfidenceLevel`               | `double`      | `0.95`                | Confidence level for the Error column                     |
| `MaxAbsoluteThresholdTolerance` | `double`      | `1.0`                 | Multiplier relaxed on shared CI runners (1.0 = no slack)  |
| `RequireIsolation`              | `bool`        | `true`                | Fail the test if measured in the host rather than a worker. Settable on `PerformanceAssertionOptions` only |

`PerformanceAssertionOptions` (NUnit / MSTest, used with the assert pattern) is an `init`-property class exposing the same properties, including a settable `RequireIsolation` (default `true`).

```csharp
[PerformanceFact(
    MaxMeanNs = 500_000,
    MaxP95Ns = 750_000,
    MaxAllocatedBytes = 4_096,
    Iterations = 500,
    WarmupIterations = 50)]
public void Serialize() => JsonSerializer.Serialize(_dto);
```

## Reference-method regression checks

`ReferenceMethod` names another method on the same test class to use as the reference baseline. The test fails if `measured.median > reference.median × MaxSlowdownRatio` (and the difference is statistically significant). The reference method must either accept the same arguments as the benchmark method or be parameterless. If `ReferenceMethod` is null and `MaxSlowdownRatio` is set, the framework uses the built-in `PerformanceCalibration` baseline (a cached one-off calibration run) instead.

```csharp
// Attribute pattern: reference another method on the same class
[PerformanceFact(MaxSlowdownRatio = 1.10, ReferenceMethod = nameof(Baseline))]
public void NewImpl() => NewParser.Parse(Payload);

[PerformanceFact]
public void Baseline() => OldParser.Parse(Payload);
```

> The assert pattern (NUnit / MSTest `PerformanceAssert`) does **not** support `ReferenceMethod` — passing a non-null value produces a hard violation. Use the attribute pattern for reference-method comparisons, or leave `ReferenceMethod` null to use the calibration baseline.

## `NBenchmark.Integration.Abstractions`

Namespace `NBenchmark.Integration.Abstractions` — the shared building blocks used by every framework package. Use these directly for custom assertions (e.g. inline in xUnit).

| Member                            | Signature                                                                                                                                            | Purpose                                                      |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `IPerformanceThresholds`          | interface, 13 props above                                                                                                                            | Common threshold contract                                    |
| `PerformanceThresholds`           | sealed `init` class — `MaxMeanNs?`, `MaxP95Ns?`, `MaxAllocatedBytes?` (all `null`), `MaxSlowdownRatio=0`, `Iterations=0`, `WarmupIterations=0`, `MaxAbsoluteThresholdTolerance=1.0` | Nullable threshold bag for `BenchmarkAssert.Validate` (only the absolute thresholds; does not implement `IPerformanceThresholds`) |
| `BenchmarkAssert.Validate`        | `(BenchmarkResult, PerformanceThresholds) → IReadOnlyList<string>`                                                                                   | Returns violation messages (checks mean / P95 / allocations) |
| `BenchmarkAssert.SetHostAssessment` / `ResetHostAssessment` | `(HostAssessment) → void` / `() → void`                                                                                                 | Override the shared-runner detection used to relax absolute thresholds |
| `MeasurementOptionsBuilder.Build` | `(IPerformanceThresholds) → MeasurementOptions`                                                                                                      | Translate thresholds into measurement options                |
| `RelativeComparison.Check`        | `(BenchmarkResult candidate, double[] candidateSamples, BenchmarkResult reference, double[] referenceSamples, double maxSlowdownRatio, double significanceLevel = 0.05) → IReadOnlyList<string>` | Pairwise regression check (Mann-Whitney U + ratio gate) |
| `RelativeComparison.CheckStructured` | Same args → `RelativeComparisonVerdict` (violations + ratio, p-value, Cliff's δ, IsRegression)                                                   | Structured form of `Check`                                   |
| `PerformanceCalibration.Run` / `CreateBenchmarkResult` | `() → CalibrationResult` (cached) / `() → BenchmarkResult`                                                                                  | Built-in fallback baseline when `ReferenceMethod` is null and `MaxSlowdownRatio` is set |
| `MetricsFormatter.Format`         | `(BenchmarkResult) → string`                                                                                                                         | Human-readable metrics block                                 |

Inline assertion in **xUnit** (no `PerformanceAssert` there):

```csharp
using NBenchmark;
using NBenchmark.Integration.Abstractions;

[Fact]
public void Query_Is_Fast()
{
    var result = Benchmark.Run(() => repo.GetRecentOrders(100), name: "GetRecentOrders");
    var violations = BenchmarkAssert.Validate(
        result, new PerformanceThresholds { MaxMeanNs = 2_000_000 });

    Assert.True(violations.Count == 0, string.Join(Environment.NewLine, violations));
}
```

## Per-framework specifics

### xUnit (`NBenchmark.Integration.xUnit`)

- `[PerformanceFact]` (`: FactAttribute`) and `[PerformanceTheory]` (`: TheoryAttribute`), both implement `IPerformanceThresholds`.
- `PerformanceAssertException : Exception` is thrown on violations (surfaced by the discoverers as a failed test).
- No `PerformanceAssert` class — use `BenchmarkAssert.Validate` for inline checks.
- Supported method return types: `void`, `Task`, `ValueTask`, `Task<T>`, `ValueTask<T>`, or any sync `T` (wrapped so the JIT can't elide the return value).

### NUnit (`NBenchmark.Integration.NUnit`)

- `[Performance]` (`: NUnitAttribute`, targets `Method`) implements `IPerformanceThresholds`. Implements NUnit's `ISimpleTestBuilder`, `IWrapTestMethod`, `IApplyToTest`.
- `PerformanceAssert` static methods: `Run(Action, PerformanceAssertionOptions? = null, string name = "Benchmark", CancellationToken = default)`, `Run<T>(Func<T> …)`, `RunAsync(Func<Task> …)`, `RunAsync<T>(Func<Task<T>> …)`, `Validate(BenchmarkResult, PerformanceAssertionOptions? = null)`, and `Validate(BenchmarkResult, double[] rawSamples, PerformanceAssertionOptions? = null)`. `Run*` return the `BenchmarkResult`.
- `PerformanceAssertionOptions` (init-only class) exposes the same thirteen threshold properties (including `RequireIsolation`).
- Violations fail via `Assert.Fail` (no public custom exception type).
- `ReferenceMethod` is **rejected** in the assert pattern — passing a non-null value produces a hard violation message.

### MSTest (`NBenchmark.Integration.MSTest`)

- `[PerformanceTestMethod]` (`: TestMethodAttribute`, targets `Method`) implements `IPerformanceThresholds`. Overrides `ExecuteAsync` to run the benchmark and return a one-element `TestResult[]` (`Passed` / `Failed`).
- `PerformanceAssert` and `PerformanceAssertionOptions` are identical in shape to NUnit's.
- `PerformanceAssertException : AssertFailedException` (`[Serializable]`) — extends MSTest's `AssertFailedException` so the runner treats it as a test failure.
- Same `ReferenceMethod` rejection as NUnit in the assert pattern.

All integration packages target net8.0/net9.0/net10.0. The isolation layer is the shared worker model described in [Where your test is measured](#where-your-test-is-measured) - there is no `[IsolatedProcess]` or detail-level concept in the integration layer; those are Harness/reporter features.

## `[AllowInProcessGate]`

The opt-out from the isolation requirement. Apply to a method, a class, or an assembly (`AttributeUsage = Method | Class | Assembly`, `Inherited = true`). It waives both the `RequireIsolation` failure and the ratio-gate restriction that normally applies to in-host measurements - so a test that must run in the host (it observes host state, or uses a fixture a worker cannot reproduce) can still enforce a ratio gate without silently being skipped.

```csharp
using NBenchmark.Integration.Abstractions;

[assembly: AllowInProcessGate]   // whole assembly, or scope it to a class/method
```

> There is deliberately no `RequireIsolation = false` on the attributes. xUnit reads attribute values as named arguments, and a settable `bool` could not express `false` (the default of `true` would be indistinguishable from "user set it to true"). The attribute is the opt-out; `PerformanceAssertionOptions.RequireIsolation` (assert pattern) is settable.

## Paired ratio gate (LaunchCount >= 2)

With `LaunchCount` set to two or more, the candidate and its reference are measured **co-resident in one worker per replicate**; their per-launch ratios are combined into a geometric mean with a multiplicative confidence interval. The ratio gate then turns on the interval **excluding 1.00x** rather than a pooled Mann-Whitney p-value, so a failure is a statement about reproducibility rather than sample count. An interval spanning 1.00x is inconclusive, not a pass. Single-launch behaviour is unchanged.

`LaunchCount` is spent per data row where applicable (an xUnit `[Theory]` with 8 rows and `LaunchCount = 3` runs 24 workers). See the core skill's [paired ratio estimation reference](../nbenchmark/references/significance-and-outliers.md#paired-ratio-estimation) for the estimator and what gates on it.

## Related skills

- **nbenchmark** — `Benchmark.Run`, `BenchmarkResult`, `MeasurementOptions`, worker isolation (`IsolationStatus`, runtime profiles)
- **nbenchmark-host** — `--threshold-pct` CI gate for standalone benchmark projects, `--strict-isolation`
- **nbenchmark-reporters** — JSON output used to capture baseline files
- **nbenchmark-troubleshooting** — flaky thresholds, noisy CI machines, tuning iterations, NB0014
