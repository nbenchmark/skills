---
name: nbenchmark-integration
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-12
description: NBenchmark test-framework integration. Use when the user wants to enforce performance thresholds (max mean, max P95, max allocations, baseline regression) as part of an existing xUnit, NUnit, or MSTest test suite — via [PerformanceFact]/[PerformanceTheory], [Performance], [PerformanceTestMethod], or PerformanceAssert.Run. For dedicated benchmark projects with a CLI regression gate (--threshold-pct) see nbenchmark-host instead.
---

# NBenchmark Test Integration

These packages let you assert performance budgets inside your **existing** test suite — no separate benchmark project or CI step. A test fails when the measured benchmark violates a threshold.

| Package                         | Framework      |
| ------------------------------- | -------------- |
| `NBenchmark.Integration.xUnit`  | xUnit v2       |
| `NBenchmark.Integration.NUnit`  | NUnit 3 / 4    |
| `NBenchmark.Integration.MSTest` | MSTest v2 / v3 |

All three depend on `NBenchmark` (core) and pull in the shared `NBenchmark.Integration.Abstractions` package automatically.

## When to use this skill

- Fail a unit test if code is slower than a budget
- Enforce a max allocation budget in tests
- Guard against regressions using a saved baseline file
- Measure just one part of a larger test (`PerformanceAssert.Run`)

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

The attributes implement `IPerformanceThresholds` and expose **all ten** properties. A `-1` (for `double`/`long`) means "disabled"; omitting a property is the same as `-1`.

| Property             | Type          | Default               | Description                                               |
| -------------------- | ------------- | --------------------- | --------------------------------------------------------- |
| `MaxMeanNs`          | `double`      | `-1` (disabled)       | Max allowed mean (ns)                                     |
| `MaxP95Ns`           | `double`      | `-1` (disabled)       | Max allowed P95 (ns)                                      |
| `MaxAllocatedBytes`  | `long`        | `-1` (disabled)       | Max mean bytes/op; implicitly enables allocation tracking |
| `BaselinePath`       | `string?`     | `null`                | JSON baseline file to compare against                     |
| `MaxSlowdownRatio`   | `double`      | `1.2`                 | Max slowdown vs baseline (1.2 = +20%)                     |
| `Iterations`         | `int`         | `0` (use default 200) | Measured-iteration override                               |
| `WarmupIterations`   | `int`         | `0` (use default 25)  | Warmup override                                           |
| `MeasureAllocations` | `bool`        | `false`               | Enable allocation tracking                                |
| `OutlierMode`        | `OutlierMode` | `IqrFence`            | Outlier strategy                                          |
| `ConfidenceLevel`    | `double`      | `0.95`                | Confidence level for the Error column                     |

`PerformanceAssertionOptions` (NUnit / MSTest, used with the assert pattern) is an `init`-property class exposing the same ten properties.

```csharp
[PerformanceFact(
    MaxMeanNs = 500_000,
    MaxP95Ns = 750_000,
    MaxAllocatedBytes = 4_096,
    Iterations = 500,
    WarmupIterations = 50)]
public void Serialize() => JsonSerializer.Serialize(_dto);
```

## Baseline regression checks

Save a known-good run to JSON, then point `BaselinePath` at it. The test fails if the baseline file is missing, the benchmark name isn't in the file, or `measured.median > baseline.median × MaxSlowdownRatio`.

```csharp
// 1. Capture a baseline once (e.g. a one-off run committed to the repo)
await Benchmark.Run(() => Parse(Payload), name: "ParseJson").ToJsonAsync("baselines/");

// 2. Guard against regressions in CI
[PerformanceFact(BaselinePath = "baselines/parse-json.json", MaxSlowdownRatio = 1.1)] // fail if >10% slower
public void ParseJson() => Parse(Payload);
```

## `NBenchmark.Integration.Abstractions`

Namespace `NBenchmark.Integration.Abstractions` — the shared building blocks used by every framework package. Use these directly for custom assertions (e.g. inline in xUnit).

| Member                            | Signature                                                                                                                                            | Purpose                                                      |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `IPerformanceThresholds`          | interface, 10 props above                                                                                                                            | Common threshold contract                                    |
| `PerformanceThresholds`           | sealed `init` class — `MaxMeanNs?`, `MaxP95Ns?`, `MaxAllocatedBytes?`, `BaselinePath?`, `MaxSlowdownRatio=1.2`, `Iterations=0`, `WarmupIterations=0` | Nullable threshold bag for `BenchmarkAssert`                 |
| `BenchmarkAssert.Validate`        | `(BenchmarkResult, PerformanceThresholds) → IReadOnlyList<string>`                                                                                   | Returns violation messages (checks mean / P95 / allocations) |
| `MeasurementOptionsBuilder.Build` | `(IPerformanceThresholds) → MeasurementOptions`                                                                                                      | Translate thresholds into measurement options                |
| `RegressionBaseline.Check`        | `(BenchmarkResult, string baselinePath, double maxSlowdownRatio) → IReadOnlyList<string>`                                                            | Baseline-file comparison                                     |
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

### NUnit (`NBenchmark.Integration.NUnit`)

- `[Performance]` (`: NUnitAttribute`) implements `IPerformanceThresholds`.
- `PerformanceAssert` static methods: `Run(Action, PerformanceAssertionOptions? = null, string name = "Benchmark", CancellationToken = default)`, `Run<T>(Func<T> …)`, `RunAsync(Func<Task> …)`, `RunAsync<T>(Func<Task<T>> …)`, and `Validate(BenchmarkResult, PerformanceAssertionOptions? = null)`. `Run*` return the `BenchmarkResult`.
- Violations fail via `Assert.Fail` (no public custom exception type).

### MSTest (`NBenchmark.Integration.MSTest`)

- `[PerformanceTestMethod]` (`: TestMethodAttribute`) implements `IPerformanceThresholds`.
- `PerformanceAssert` and `PerformanceAssertionOptions` are identical in shape to NUnit's.
- `PerformanceAssertException : AssertFailedException` (`[Serializable]`).

All integration packages target net8.0/net9.0/net10.0. There is no `[IsolatedProcess]` or detail-level concept in the integration layer — those are Host/reporter features.

## Related skills

- **nbenchmark** — `Benchmark.Run`, `BenchmarkResult`, `MeasurementOptions`
- **nbenchmark-host** — `--threshold-pct` CI gate for standalone benchmark projects
- **nbenchmark-reporters** — JSON output used to capture baseline files
- **nbenchmark-troubleshooting** — flaky thresholds, noisy CI machines, tuning iterations
