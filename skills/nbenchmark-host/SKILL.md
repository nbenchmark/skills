---
name: nbenchmark-host
nbenchmarkVersion: v0.37.0
lastVerified: 2026-08-01
description: NBenchmark Harness mode for dedicated benchmark projects. Use when the user wants attribute-based benchmark discovery ([Benchmark], [BenchmarkCase]/[BenchmarkCases], lifecycle attributes, [IsolatedProcess]/[InProcess]), a command-line interface (BenchmarkHarness.Create(args)), constructor dependency injection for benchmark classes, or CI/CD regression gates. Every benchmark class runs in a dedicated worker process by default. For one-off measurements and suites see the core nbenchmark skill; for output formats see nbenchmark-reporters.
---

# NBenchmark Harness Mode

`BenchmarkHarness` is for **dedicated benchmark projects** — a separate console app that scans assemblies for `[Benchmark]`-decorated methods and exposes a full CLI, so you can filter and configure runs from the terminal without recompiling.

> The CLI entry point is the `NBenchmark.Tool` package (`dotnet benchmark` global tool). It forwards unknown flags to `BenchmarkHarness.Create(args)`. Most users wire up the harness directly in their own console `Program.cs`, as shown below — `BenchmarkHarness.Create(args)` is the public API.

Install: `dotnet add package NBenchmark` (+ `NBenchmark.Reporters.Console` for terminal output). For the standalone global tool instead of a custom Program.cs: `dotnet tool install -g NBenchmark.Tool` (command name `benchmark`).

## When to use this skill

- Set up a dedicated benchmark console project
- Discover benchmarks via `[Benchmark]` attributes
- Parameterize benchmarks with `[BenchmarkCase]` / `[BenchmarkCases]`
- Use setup/teardown lifecycle hooks
- Isolate a benchmark in its own worker (`[IsolatedProcess]`) or opt out of the default (`[InProcess]`, `--in-process`)
- Drive runs from the command line
- Inject dependencies into benchmark classes
- Gate CI on performance regressions

## Minimal setup

```bash
dotnet new console -n MyApp.Benchmarks
cd MyApp.Benchmarks
dotnet add package NBenchmark
dotnet add package NBenchmark.Reporters.Console
dotnet add reference ../MyApp/MyApp.csproj
```

```csharp
// Program.cs
using NBenchmark;
using NBenchmark.Reporters.Console;
using NBenchmark.Attributes;

await BenchmarkHarness.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .WithReporter(new ConsoleReporter())
    .WithProgress(new ConsoleBenchmarkProgress())
    .RunAsync();

public class StringBenchmarks
{
    [Benchmark(Baseline = true)]
    public string Concat() => "hello" + " " + "world";

    [Benchmark]
    public string Interpolate() => $"hello {"world"}";
}
```

```bash
dotnet run
dotnet run -- --filter String*
dotnet run -- --reporter markdown --output ./results
```

> Attributes live in the `NBenchmark.Attributes` namespace. The rich console types (`ConsoleReporter`, `ConsoleBenchmarkProgress`) live in `NBenchmark.Reporters.Console`.

## Attributes

All attributes are in `NBenchmark.Attributes`.

### `[Benchmark]`

Marks a **public instance** method (sync, `Task`, or `Task<T>`). Return a value to prevent dead-code elimination.

| Property           | Type      | Default      | Description                               |
| ------------------ | --------- | ------------ | ----------------------------------------- |
| `Description`      | `string?` | `null`       | Optional label shown in output            |
| `Baseline`         | `bool`    | `false`      | Marks the baseline for ratio/significance |
| `Iterations`       | `int`     | `-1` (unset) | Per-method iteration override (0–100,000) |
| `WarmupIterations` | `int`     | `-1` (unset) | Per-method warmup override (0–10,000)     |
| `LaunchCount`      | `int`     | `-1` (unset) | Per-method launch-count override (1–100)  |

`Iterations`/`WarmupIterations`/`LaunchCount` use `-1` as the "unset" sentinel (named attribute arguments can't be nullable value types). Set a value ≥ 0 to override the host options for that method only. Helpers `HasIterationsOverride` / `HasWarmupIterationsOverride` / `HasLaunchCountOverride` report whether an override is present.

`OpsPerSample` is **not** a per-method attribute argument — pin it host-wide with `.WithOpsPerSample(n)` or `--ops-per-sample n`. Only `Iterations` and `WarmupIterations` are pinnable per method.

```csharp
[Benchmark(Baseline = true, Description = "current production")]
public string CurrentImpl() => Production.DoWork();

[Benchmark(Iterations = 1000, WarmupIterations = 100)]
public string HotPath() => Candidate.DoWork();
```

### `[BenchmarkCase(params object[])]` and `[BenchmarkCases(string sourceName)]`

Two complementary attributes for parameterised benchmarks (`AllowMultiple = true` on `[BenchmarkCase]`). The method must accept matching parameters; each case becomes a separate row named `MethodName(arg1, arg2, ...)`. Argument count/type mismatches are caught by analyzer NB0003; combining both attributes on one method is NB0012.

`[BenchmarkCase(...)]` is for a short inline list of literal arguments:

```csharp
[Benchmark(Baseline = true)]
[BenchmarkCase(10)]
[BenchmarkCase(1_000)]
[BenchmarkCase(100_000)]
public void Sort(int n)
{
    var arr = Enumerable.Range(0, n).Reverse().ToArray();
    Array.Sort(arr);
}
```

`[BenchmarkCases(nameof(Source))]` is for programmatic, named, or generated cases. The source method must be parameterless (static or instance), return `IEnumerable<ValueTuple<...>>`, and have an arity matching the benchmark method's parameters (max 7). Tuple element names become columns in the report:

```csharp
[Benchmark(Baseline = true)]
[BenchmarkCases(nameof(BinarySearchCases))]
public int BinarySearch(int count, string targetLabel) { /* ... */ }

public static IEnumerable<(int Count, string Target)> BinarySearchCases()
{
    yield return (100, "first");
    yield return (10000, "middle");
    yield return (100000, "last");
}
```

### Lifecycle attributes

All must be **parameterless** methods. Duplicate lifecycle attributes in one class are an error (NB0007).

| Attribute                      | Runs                                   | Measured? |
| ------------------------------ | -------------------------------------- | --------- |
| `[BenchmarkSetup]`             | Once before any benchmark in the class | No        |
| `[BenchmarkTeardown]`          | Once after all benchmarks in the class | No        |
| `[BenchmarkIterationSetup]`    | Before each iteration                  | No        |
| `[BenchmarkIterationTeardown]` | After each iteration                   | No        |

```csharp
public class DatabaseBenchmarks
{
    private DbConnection _conn = null!;

    [BenchmarkSetup] public void Open() => _conn = new DbConnection(cs);
    [BenchmarkTeardown] public void Close() => _conn.Dispose();
    [BenchmarkIterationSetup] public void Begin() => _conn.BeginTransaction();
    [BenchmarkIterationTeardown] public void Rollback() => _conn.RollbackTransaction();

    [Benchmark] public void RunQuery() => _conn.Execute("SELECT COUNT(*) FROM orders");
}
```

### `[IsolatedProcess]` / `[InProcess]`

Isolation is the **default**: every benchmark class runs in its own freshly spawned worker process. `[IsolatedProcess]` and `[InProcess]` are the per-method/per-class granular controls on top of that default. Apply either to a method, or to a class to affect every benchmark it declares (`AttributeUsage = Method | Class`, `Inherited = true`).

| Attribute | Effect |
|---|---|
| `[IsolatedProcess]` | The benchmark gets its **own dedicated worker** (finest granularity - one worker per method, not one per class). Use it when a benchmark is sensitive to runtime warmup state from siblings. |
| `[InProcess]` | Forces in-process execution for that method/class, even though the default is isolated. The opt-out at the method/class scope. |

```csharp
public class StartupBenchmarks
{
    [Benchmark]
    [IsolatedProcess]
    public int ColdPath() => RunColdSensitiveWork();
}
```

Each isolated benchmark runs in a clean CLR, so it isn't influenced by JIT, GC, or thread-pool state warmed up by sibling benchmarks. The harness launches a dedicated `nbworker` executable, sends the work over a binary protocol, and reads results back (never via stdout, so the worker's console output can't corrupt the data). The whole-run opt-outs are `--in-process` / `WithIsolation(false)`; `--dry-run` is always in-process. In-process and isolated benchmarks coexist in one suite and run separately - results carry an `IsolationStatus` and the reporter shows an `Iso` column in mixed-isolation tables.

A body that captures local state falls back to the host with a labelled reason (`IsolationStatus.InProcessCapturedState`); use `Benchmark.Run(prepare:, body:)` / `BenchmarkSuite.WithState(...)` to hand the worker a recipe instead of a value. See the core skill's [isolation reference](../nbenchmark/references/isolation.md) for the worker model, runtime profiles, and the capture fallback.

### `[InstanceLifetime]`

Controls whether the harness reuses one instance across all `[Benchmark]` methods in a class (`PerClass`, the default) or creates a fresh instance per method (`PerMethod`). Apply to a class only.

```csharp
[InstanceLifetime(InstanceLifetime.PerMethod)]
public class StatelessBenchmarks { /* ... */ }
```

With `PerClass` and multiple `[Benchmark]` methods sharing mutable state, the analyzers NB0011 (scoped service injection) and NB0013 (mutable instance field) warn about possible state contamination. Implement `NBenchmark.Lifecycle.IStateReset` (a `ResetAsync(CancellationToken)` method) to opt out of NB0011 by making the contamination explicit and resettable.

### `[Runtimes]`

Declares target runtimes for cross-runtime comparison (`RuntimeMoniker.Net8` / `Net9` / `Net10`). The harness builds and runs each runtime in its own worker process. Suite-mode multi-runtime requires `[BenchmarkPlan]`. Apply to a class only.

```csharp
[Runtimes(RuntimeMoniker.Net8, RuntimeMoniker.Net10)]
public class CrossRuntimeBenchmarks { /* ... */ }
```

### `[BenchmarkCategory]`

Tags a benchmark method or class with a category (`AllowMultiple = true`, `Method | Class`). Use with `--filter`/`--category-include`/`--category-exclude` or `WithCategoryFilter` to scope runs.

```csharp
[BenchmarkCategory("cold")]
[Benchmark]
public int ColdPath() => RunColdSensitiveWork();
```

## Class requirements

Classes are instantiated via `Activator.CreateInstance`, so they need a **public parameterless constructor** (NB0001 warns if missing). For constructor dependencies, use the DI companion package below.

## Fluent host configuration

`BenchmarkHarness.Create(args)` returns a builder:

| Method                                               | Purpose                                                                    |
| ---------------------------------------------------- | -------------------------------------------------------------------------- |
| `AddFromAssembly<T>()` / `AddFromAssembly(Assembly)` | Scan an assembly for `[Benchmark]` methods (call once per assembly)        |
| `WithReporter(reporter)`                             | Add an `IReporter` (stackable)                                             |
| `WithOptions(MeasurementOptions)`                    | Default measurement options (CLI flags override)                           |
| `WithLaunchCount(n)`                                 | Repeat the suite as `n` separate launches (default 1)                       |
| `WithRunOrder(order)`                                | `RunOrder.Random` (default) or `RunOrder.Declaration`                      |
| `WithDetail(detail)`                                 | `ReportDetail.Simple` (default) / `Standard` / `Advanced`                  |
| `WithProgress(progress)`                             | Live progress callback (`ConsoleBenchmarkProgress` in the console package) |
| `WithObserver(observer)`                             | Non-perturbing `IMeasurementObserver` telemetry                            |
| `WithInstanceFactory(factory)`                        | Custom factory for benchmark class instances (DI hook)                     |
| `WithServiceProvider(sp)`                             | Resolve benchmark instances from an `IServiceProvider` (DI package)         |
| `WithMinimumPracticalEffect(delta)`                  | Downgrade Sig to NotSignificant below this Cliff's δ effect size            |
| `WithHardwareAffinity(params int[] cores)`           | Pin CPU cores for the run                                                  |
| `WithProcessPriority(priority)`                       | Set process priority                                                        |
| `WithDedicatedHostGuidance(bool = true)`             | Warn if running on a shared CI host                                          |
| `WithMeasurementProfile(profile)`                    | `Realistic` (default) or `Independent` GC behaviour                         |
| `WithAutoTune(options)` / `WithAutoTune(preset)`     | Bound/steer the adaptive loop                                               |
| `WithOpsPerSample(n)`                                | Pin K — body invocations per timed sample                                   |
| `WithDiagnostics(options)` / `WithDiagnostics(mode)` | GC/heap/exception/CPU-time diagnostics                                       |
| `WithIsolation(bool = true)`                         | Superseded - isolation is the default. `WithIsolation(false)` opts back into the host, deliberately and silently |
| `WithRuntimeProfile(profile)`                        | Runtime-startup config (`SteadyState`/`Production`/`ServerGc`/`Host`); default `SteadyState` |
| `WithCrossClassSignificance(bool = true)`            | Run significance tests across classes (default false - within-class only)   |
| `WithInstanceLifetime(lifetime)`                     | `PerClass` (default) or `PerMethod` instance reuse                          |
| `WithCategoryFilter(include?, exclude?)`             | Run only benchmarks matching the category filter                            |
| `RunAsync()`                                         | Parse args, discover, run; returns `IReadOnlyList<BenchmarkResult>`        |

```csharp
await BenchmarkHarness.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .AddFromAssembly<DatabaseBenchmarks>()
    .WithOptions(new MeasurementOptions
    {
        Iterations = 500,
        WarmupIterations = 50,
        ConfidenceLevel = 0.99,
    })
    .WithDetail(ReportDetail.Advanced)
    .WithReporter(new ConsoleReporter())
    .RunAsync();
```

Significance is configured through `WithOptions` (`EnableSignificance`, `SignificanceLevel`, `MinimumPracticalEffect`) or the `--alpha` CLI flag — there is no `WithSignificance` directly on the host. CLI flags always override `WithOptions` values.

## Command-line interface

`BenchmarkHarness.Create(args)` parses arguments automatically — no parsing library needed.

```bash
dotnet run -- --filter Sort* --iterations 1000 --reporter markdown --output ./results
dotnet run -- --list        # discover without running
dotnet run -- --dry-run     # wire up everything, never invoke the body
dotnet run -- --detail advanced
dotnet run -- --in-process  # opt out of worker isolation; measure in this process
dotnet run -- --strict-isolation  # fail if any benchmark could not be isolated
dotnet run -- --runtime-profile production  # measure under tiered+PGO+R2R
```

Frequently used flags: `--filter`, `--iterations`, `--warmup`, `--auto-tune`, `--ops-per-sample`, `--ci-target`, `--min-samples`, `--max-samples`, `--min-warmup`, `--max-warmup`, `--max-tuning-time`, `--confidence`, `--alpha`, `--reporter`, `--output`, `--order`, `--seed`, `--detail`, `--list`, `--dry-run`, `--in-process`, `--strict-isolation`, `--verify-isolation`, `--runtime-profile`, `--stream-samples`, `--emit-raw`, `--launch-count`, `--profile`, `--force-gc`, `--no-allocations`, `--threshold-pct`, `--help`/`-h`.

If you use the standalone `NBenchmark.Tool` global tool (`dotnet benchmark`), it also accepts `--project <path>` (builds a .csproj with `dotnet build -c Release` and benchmarks the resulting DLL) and `--assembly <path>` (benchmarks a pre-built DLL). With neither, it auto-discovers `*.dll` in the current directory. All other flags are forwarded to `BenchmarkHarness.Create`.

See [references/cli.md](references/cli.md) for every flag, exit codes, and CI examples.

## Dependency injection

Add `NBenchmark.DependencyInjection` to give benchmark classes constructor dependencies. Any `IServiceProvider` works (Microsoft DI, Autofac, etc.).

```csharp
using Microsoft.Extensions.DependencyInjection;
using NBenchmark.DependencyInjection;

var services = new ServiceCollection()
    .AddSingleton<IOrderRepository, SqlOrderRepository>()
    .AddTransient<OrderBenchmarks>()
    .BuildServiceProvider();

await BenchmarkHarness.Create(args)
    .UseDependencyInjection<OrderBenchmarks>(services)   // = AddFromAssembly<T>().WithServiceProvider(sp)
    .RunAsync();

public sealed class OrderBenchmarks(IOrderRepository repository)
{
    [Benchmark]
    public int CountOrders() => repository.Count();
}
```

DI extension methods:

| Method                                | Behaviour                                                                                       |
| ------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `WithServiceProvider(sp)`             | Resolve benchmark instances from `sp` (root lifetime - instances not disposed by the harness)   |
| `WithScopedServiceProvider(sp)`       | Create a DI scope per benchmark-class instance; disposed after `[BenchmarkTeardown]`           |
| `UseDependencyInjection<T>(sp)`       | `AddFromAssembly<T>()` + `WithServiceProvider(sp)`                                              |
| `UseScopedDependencyInjection<T>(sp)` | `AddFromAssembly<T>()` + `WithScopedServiceProvider(sp)`                                        |

## CI/CD regression gate

`--threshold-pct <n>` exits with code **1** if any benchmark's median is more than `n`% slower than the baseline. Combine with a file reporter to keep the evidence (reporters still flush on a threshold failure).

```bash
dotnet run -c Release -- --threshold-pct 10 --reporter json --output ./results
```

```yaml
# GitHub Actions
- name: Run benchmarks
  run: dotnet run -c Release --project benchmarks -- --threshold-pct 10 --reporter markdown --output ./bench
```

For richer assertions (allocation budgets, P95 limits, baseline files) inside your existing test suite, see the `nbenchmark-integration` skill.

## Telemetry (OpenTelemetry / OTLP)

NBenchmark emits a full set of OpenTelemetry metrics and traces through a `Meter` / `ActivitySource` (name `"NBenchmark"`). Pass `--otlp-endpoint <url>` to export them to a collector; the endpoint is forwarded to workers so they stream to the same collector as the host.

```bash
dotnet run -c Release -- --otlp-endpoint http://localhost:4317
```

Instruments include per-sample histograms (`nbenchmark.sample.duration`, `nbenchmark.alloc.bytes_per_op`), GC and outlier counters, live gauges (CI half-width, jitter metric, ops/s), and a three-level trace (`benchmark.suite` -> `benchmark.run` -> `nbenchmark.phase.{phase}`) with span events for detector switches, warmup plateau, CI target met, and cap hits. Resource attributes (commit SHA, branch, CI provider, host) are stamped on the root span from environment variables.

See [references/telemetry.md](references/telemetry.md) for every instrument name, tag, span, span event, and resource attribute.

## References

- [cli.md](references/cli.md) - every CLI flag, exit codes, CI examples
- [telemetry.md](references/telemetry.md) - OpenTelemetry instruments, spans, resource attributes
- [isolation.md](../nbenchmark/references/isolation.md) - worker model, runtime profiles, `IsolationStatus`, `--strict-isolation`/`--verify-isolation` (cross-cutting)

## Related skills

- **nbenchmark** — Single and Suite modes, `MeasurementOptions`, `BenchmarkResult`, worker isolation
- **nbenchmark-reporters** — reporter pipeline, detail levels, custom reporters, observers
- **nbenchmark-integration** — performance thresholds as xUnit/NUnit/MSTest tests
- **nbenchmark-troubleshooting** — analyzer diagnostics (NB0001-NB0014), wrong results, tuning
