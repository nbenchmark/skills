---
name: nbenchmark-host
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-09
description: Guide for using NBenchmark's Host mode for dedicated benchmark projects. Use when the user wants to set up a benchmark project with attribute-based discovery, CLI flags, dependency injection, or CI/CD regression detection with --threshold-pct.
---

# NBenchmark Host Mode

Host mode provides a dedicated benchmark runner using reflection-based discovery, attributes, and a built-in CLI. It is designed for standalone benchmark projects.

## When to use this skill

User asks to:

- Create a dedicated benchmark project
- Use `[Benchmark]` and related attributes
- Run benchmarks from the command line with `--filter`, `--list`, `--dry-run`
- Add dependency injection to benchmarks
- Set up CI/CD regression detection
- Scan multiple assemblies for benchmarks

## Project Setup

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="NBenchmark" Version="*" />
    <!-- Optional: rich console output -->
    <PackageReference Include="NBenchmark.Console" Version="*" />
  </ItemGroup>
</Project>
```

## Basic program.cs

```csharp
using NBenchmark;

await BenchmarkHost.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .WithReporter(new ConsoleReporter()) // Requires NBenchmark.Console
    .RunAsync();

public class StringBenchmarks
{
    [Benchmark(Baseline = true)]
    public string Concat() => "hello" + " " + "world";

    [Benchmark]
    public string Interpolate() => $"hello {"world"}";
}
```

## Attributes reference

### [Benchmark]

```csharp
[Benchmark]
[Benchmark(Baseline = true)]
[Benchmark(Description = "My benchmark")]
[Benchmark(Iterations = 500, WarmupIterations = 50)]
[Benchmark(Baseline = true, Description = "current impl")]
```

Properties: `Description` (string?), `Baseline` (bool), `Iterations` (int, sentinel -1 = unset), `WarmupIterations` (int, sentinel -1 = unset). Per-method `Iterations`/`WarmupIterations` override the suite-level options. Sentinel value `HasIterationsOverride` / `HasWarmupIterationsOverride` computed properties.

### [BenchmarkArguments]

```csharp
[BenchmarkArguments("hello", 42)]
[BenchmarkArguments("world", 99)]
public string ConcatWith(string prefix, int count)
    => string.Concat(Enumerable.Repeat(prefix, count));
```

Allows multiple per method; each set creates a separate benchmark entry. Arguments are converted via `Convert.ChangeType` with invariant culture. Nullable value types (e.g. `int?`) are unwrapped before conversion. Display name is auto-generated as `MethodName(arg1, arg2, ...)`.

### Lifecycle attributes

| Attribute                      | When it runs                              | Count         |
| ------------------------------ | ----------------------------------------- | ------------- |
| `[BenchmarkSetup]`             | Once before any benchmark in the class    | 1             |
| `[BenchmarkTeardown]`          | Once after all benchmarks in the class    | 1             |
| `[BenchmarkIterationSetup]`    | Before each warmup and measured iteration | Per iteration |
| `[BenchmarkIterationTeardown]` | After each warmup and measured iteration  | Per iteration |

Only one method per lifecycle attribute type is used per class (first found by reflection).

## CLI flags reference

| Flag                  | What it does                                                              |
| --------------------- | ------------------------------------------------------------------------- |
| `--filter <pattern>`  | Glob filter on `ClassName.BenchmarkName` (e.g. `String*`, `*.Contains*`)  |
| `--iterations <n>`    | Override measured iterations (0 - 100,000)                                |
| `--warmup <n>`        | Override warmup iterations (0 - 10,000)                                   |
| `--reporter <type>`   | Set reporter: json, markdown, csv (or console with NBenchmark.Console)    |
| `--output <dir>`      | Output directory for file-based reporters                                 |
| `--confidence <0-1>`  | Confidence level for interval on the mean (e.g. 0.95, 0.99)               |
| `--list`              | List discovered benchmarks without running                                |
| `--dry-run`           | Run with 0 iterations; no measurement, no body invocation                 |
| `--order <mode>`      | Run order: random (default) or declaration                                |
| `--seed <n>`          | Seed for deterministic random ordering                                    |
| `--threshold-pct <n>` | Fail with exit code 1 if any benchmark regresses >N% (median-based, n>=1) |
| `--help, -h`          | Show help text                                                            |

### CLI usage examples

```bash
# List benchmarks
dotnet run -- --list

# Filter to specific benchmarks
dotnet run -- --filter String*

# Validate discovery without running
dotnet run -- --dry-run

# Set iterations and confidence
dotnet run -- --iterations 1000 --confidence 0.99

# Output to files
dotnet run -- --reporter markdown --reporter json --output ./results

# Regression check: fail if any is >10% slower than baseline
dotnet run -- --threshold-pct 10
```

### How CLI overrides work

`--iterations`, `--warmup`, `--confidence` override the programmatic `WithOptions` values only for those specific fields. All other options keep their programmatic values. The override happens via `MergeCliOptions` in `BenchmarkHost.RunAsync`.

## Assembly scanning

```csharp
// Single assembly
await BenchmarkHost.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .RunAsync();

// Multiple assemblies
await BenchmarkHost.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .AddFromAssembly<DatabaseBenchmarks>()
    .AddFromAssembly(typeof(NetworkBenchmarks).Assembly)
    .RunAsync();
```

Host scans all non-abstract types in registered assemblies for methods with `[Benchmark]`. Discovery uses instance methods only (public or non-public) - static methods are not supported and trigger analyzer NB0002.

## Class requirements for host mode

- Must have a public parameterless constructor (or use DI/`WithInstanceFactory`)
- Cannot be abstract
- Instance methods only (no static `[Benchmark]` methods)
- `[Benchmark]` method parameters require `[BenchmarkArguments]`
- `[BenchmarkArguments]` on parameterless methods raises an error
- Parameterless methods without `[BenchmarkArguments]` are allowed

## Fluent configuration

```csharp
await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .WithOptions(new MeasurementOptions
    {
        Iterations = 500,
        WarmupIterations = 50,
        OutlierMode = OutlierMode.IqrFence,
    })
    .WithRunOrder(RunOrder.Declaration)
    .WithReporter(new ConsoleReporter())
    .WithReporter(new MarkdownReporter("results.md"))
    .WithProgress(new ConsoleBenchmarkProgress(500, 50))
    .RunAsync();
```

## Progress display

```csharp
await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .WithProgress(new ConsoleBenchmarkProgress(500, 25)) // 500 measured, 25 warmup
    .RunAsync();
```

The `IBenchmarkProgress` interface has six callbacks:

1. `OnSuiteStarting` - before any benchmark runs
2. `OnWarmupStarting` - per-benchmark warmup start
3. `OnWarmupCompleted` - per-benchmark warmup end
4. `OnBenchmarkStarting` - per-benchmark measurement start
5. `OnBenchmarkCompleted` - per-benchmark measurement end
6. `OnSuiteCompleted` - after all benchmarks

## Dependency Injection

Requires `NBenchmark.DependencyInjection` package.

```csharp
using Microsoft.Extensions.DependencyInjection;
using NBenchmark.DependencyInjection;

var services = new ServiceCollection()
    .AddSingleton<IOrderRepository, SqlOrderRepository>()
    .AddTransient<OrderBenchmarks>()
    .BuildServiceProvider();

await BenchmarkHost.Create(args)
    .UseDependencyInjection<OrderBenchmarks>(services)
    .WithReporter(new ConsoleReporter())
    .RunAsync();

public sealed class OrderBenchmarks(IOrderRepository repository)
{
    [Benchmark]
    public int CountOrders() => repository.Count();
}
```

### Extension methods

| Method                                | Behaviour                                                                   |
| ------------------------------------- | --------------------------------------------------------------------------- |
| `UseDependencyInjection<T>(sp)`       | Scans assembly for T, resolves instances from `sp.GetRequiredService`       |
| `UseScopedDependencyInjection<T>(sp)` | Same, but creates a scope per suite class and disposes it after teardown    |
| `WithServiceProvider(sp)`             | Sets `WithInstanceFactory(sp.GetRequiredService)` without assembly scanning |
| `WithScopedServiceProvider(sp)`       | Sets scoped instance factory without assembly scanning                      |

### Scoped lifetime example

```csharp
await BenchmarkHost.Create(args)
    .UseScopedDependencyInjection<ScopedBenchmarks>(services)
    .RunAsync();
```

Creates one scope per class, resolving all benchmark instances from that scope. The scope is disposed after teardown.

### Custom instance factory (escape hatch)

```csharp
await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .WithInstanceFactory(type =>
    {
        if (type == typeof(MyBenchmarks))
            return new MyBenchmarks("custom-arg");
        return Activator.CreateInstance(type)!;
    })
    .RunAsync();
```

## Return value

```csharp
IReadOnlyList<BenchmarkResult> results = await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .RunAsync();

// Check exit code for CI
var hasRegression = Environment.ExitCode == 1;
```

## CI/CD Integration

### Regression threshold

```bash
# Fail CI pipeline if any benchmark is >5% slower than baseline
dotnet run -- --threshold-pct 5
```

Uses median-based comparison: `candidate.Median > baseline.Median * (1 + thresholdPct / 100)`. Sets `Environment.ExitCode = 1` on regression.

### JSON output for pipeline consumption

```bash
dotnet run -- --reporter json --output ./benchmark-results
```

### Markdown for PR comments

```bash
dotnet run -- --reporter markdown --output ./benchmark-results
```
