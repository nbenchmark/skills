---
name: nbenchmark-host
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-12
description: NBenchmark Host mode for dedicated benchmark projects. Use when the user wants attribute-based benchmark discovery ([Benchmark], [BenchmarkArguments], lifecycle attributes, [IsolatedProcess]), a command-line interface (BenchmarkHost.Create(args)), constructor dependency injection for benchmark classes, or CI/CD regression gates. For one-off measurements and suites see the core nbenchmark skill; for output formats see nbenchmark-reporters.
---

# NBenchmark Host Mode

`BenchmarkHost` is for **dedicated benchmark projects** — a separate console app that scans assemblies for `[Benchmark]`-decorated methods and exposes a full CLI, so you can filter and configure runs from the terminal without recompiling.

Install: `dotnet add package NBenchmark` (+ `NBenchmark.Reporters.Console` for terminal output).

## When to use this skill

- Set up a dedicated benchmark console project
- Discover benchmarks via `[Benchmark]` attributes
- Parameterize benchmarks with `[BenchmarkArguments]`
- Use setup/teardown lifecycle hooks
- Isolate a benchmark in its own process (`[IsolatedProcess]`)
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

await BenchmarkHost.Create(args)
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

`Iterations`/`WarmupIterations` use `-1` as the "unset" sentinel (named attribute arguments can't be nullable value types). Set a value ≥ 0 to override the host options for that method only. Helpers `HasIterationsOverride` / `HasWarmupIterationsOverride` report whether an override is present.

`OpsPerSample` is **not** a per-method attribute argument — pin it host-wide with `.WithOpsPerSample(n)` or `--ops-per-sample n`. Only `Iterations` and `WarmupIterations` are pinnable per method.

```csharp
[Benchmark(Baseline = true, Description = "current production")]
public string CurrentImpl() => Production.DoWork();

[Benchmark(Iterations = 1000, WarmupIterations = 100)]
public string HotPath() => Candidate.DoWork();
```

### `[BenchmarkArguments(params object[])]`

Runs the method once per argument set (`AllowMultiple = true`). The method must accept matching parameters. Each set becomes a separate entry named `MethodName(arg1, arg2, ...)`. Argument count/type mismatches are caught by analyzer NB0003.

```csharp
[BenchmarkArguments(10)]
[BenchmarkArguments(1_000)]
[BenchmarkArguments(100_000)]
[Benchmark]
public void Sort(int n)
{
    var arr = Enumerable.Range(0, n).Reverse().ToArray();
    Array.Sort(arr);
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

### `[IsolatedProcess]`

Runs a benchmark in a freshly spawned child process instead of in-process. Apply to a method, or to a class to isolate every benchmark it declares (`AttributeUsage = Method | Class`, `Inherited = true`).

```csharp
public class StartupBenchmarks
{
    [Benchmark]
    [IsolatedProcess]
    public int ColdPath() => RunColdSensitiveWork();
}
```

Each isolated benchmark runs in a clean CLR, so it isn't influenced by JIT, GC, or thread-pool state warmed up by sibling benchmarks. The host re-runs the entry assembly for the child, executes only that one benchmark, and reads the result back through a temporary file (never stdout, so the child's console output can't corrupt the data). It uses internal `--nb-isolated-run` / `--nb-isolated-output` flags you never pass yourself. Isolation trades a process launch per benchmark for measurement cleanliness; use it only when a benchmark is sensitive to runtime warmup state. In-process and isolated benchmarks coexist in one suite and run separately.

## Class requirements

Classes are instantiated via `Activator.CreateInstance`, so they need a **public parameterless constructor** (NB0001 warns if missing). For constructor dependencies, use the DI companion package below.

## Fluent host configuration

`BenchmarkHost.Create(args)` returns a builder:

| Method                                               | Purpose                                                                    |
| ---------------------------------------------------- | -------------------------------------------------------------------------- |
| `AddFromAssembly<T>()` / `AddFromAssembly(Assembly)` | Scan an assembly for `[Benchmark]` methods (call once per assembly)        |
| `WithReporter(reporter)`                             | Add an `IReporter` (stackable)                                             |
| `WithOptions(MeasurementOptions)`                    | Default measurement options (CLI flags override)                           |
| `WithRunOrder(order)`                                | `RunOrder.Random` (default) or `RunOrder.Declaration`                      |
| `WithDetail(detail)`                                 | `ReportDetail.Simple` (default) or `.Advanced`                             |
| `WithProgress(progress)`                             | Live progress callback (`ConsoleBenchmarkProgress` in the console package) |
| `WithInstanceFactory(factory)`                       | Custom factory for benchmark class instances (DI hook)                     |
| `RunAsync()`                                         | Parse args, discover, run; returns `IReadOnlyList<BenchmarkResult>`        |

```csharp
await BenchmarkHost.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .AddFromAssembly<DatabaseBenchmarks>()
    .WithOptions(new MeasurementOptions
    {
        Iterations = 500,
        WarmupIterations = 50,
        MeasureAllocations = true,
        ConfidenceLevel = 0.99,
    })
    .WithDetail(ReportDetail.Advanced)
    .WithReporter(new ConsoleReporter())
    .RunAsync();
```

Significance is configured through `WithOptions` (`EnableSignificance`, `SignificanceLevel`) or the `--alpha` CLI flag — there is no `WithSignificance` directly on the host. CLI flags always override `WithOptions` values.

## Command-line interface

`BenchmarkHost.Create(args)` parses arguments automatically — no parsing library needed.

```bash
dotnet run -- --filter Sort* --iterations 1000 --reporter markdown --output ./results
dotnet run -- --list        # discover without running
dotnet run -- --dry-run     # wire up everything, never invoke the body
dotnet run -- --detail advanced
```

Frequently used flags: `--filter`, `--iterations`, `--warmup`, `--auto-tune`, `--ops-per-sample`, `--ci-target`, `--min-samples`, `--max-samples`, `--confidence`, `--alpha`, `--reporter`, `--output`, `--order`, `--seed`, `--detail`, `--list`, `--dry-run`, `--threshold-pct`, `--help`/`-h`.

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

await BenchmarkHost.Create(args)
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
| `WithServiceProvider(sp)`             | Resolve benchmark instances from `sp`                                                           |
| `WithScopedServiceProvider(sp)`       | Create a DI scope for the run; disposed during post-suite cleanup (`DbContext`-style lifetimes) |
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

## Related skills

- **nbenchmark** — Quick and Suite modes, `MeasurementOptions`, `BenchmarkResult`
- **nbenchmark-reporters** — reporter pipeline, detail levels, custom reporters
- **nbenchmark-integration** — performance thresholds as xUnit/NUnit/MSTest tests
- **nbenchmark-troubleshooting** — analyzer diagnostics, wrong results, tuning
