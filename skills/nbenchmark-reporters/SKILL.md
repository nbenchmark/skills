---
name: nbenchmark-reporters
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-09
description: Guide for NBenchmark's reporter pipeline - outputting benchmark results as console tables, JSON, Markdown, CSV, or custom formats. Use when the user wants to save results to files, display rich console output, or build a custom reporter.
---

# NBenchmark Reporters

Reporters are plug-ins that write `BenchmarkResult` collections to a sink. All built-in reporters implement `IReporter` and are registered in the `ReporterRegistry`.

## When to use this skill

User asks to:

- Output benchmark results to a file (JSON, Markdown, CSV)
- Display results as a console table
- Create a custom reporter
- Use reporters with Suite mode or Host mode
- Configure CLI reporter flags
- Register a new reporter globally

## IReporter interface

```csharp
public interface IReporter
{
    string Name { get; }
    Task ReportAsync(IReadOnlyList<BenchmarkResult> results, CancellationToken ct = default);
}
```

## Attaching reporters

### Suite mode

```csharp
var results = await new BenchmarkSuite("MySuite")
    .Add("method1", () => Method1())
    .Add("method2", () => Method2())
    .WithReporter(new JsonReporter("./results"))
    .WithReporter(new MarkdownReporter("results.md"))
    .WithReporter(new CsvReporter("results.csv"))
    .RunAsync();
```

### Host mode

```csharp
await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .WithReporter(new ConsoleReporter())
    .WithReporter(new MarkdownReporter("results.md"))
    .RunAsync();
```

### Host mode via CLI

```bash
dotnet run -- --reporter json --reporter markdown --output ./results
```

CLI reporters are added before programmatic `WithReporter` reporters and are run first.

### Quick mode (single result via extensions)

```csharp
var result = Benchmark.Run(() => MyMethod());
await result.ToMarkdownAsync("benchmark.md");
await result.ToJsonAsync("./output");
await result.ToCsvAsync("benchmark.csv");
await result.PrintAsync(); // Requires NBenchmark.Console
```

## ReporterRegistry

The `ReporterRegistry` is the global registry for constructing reporters by name.

```csharp
// Check available reporters
foreach (var info in ReporterRegistry.Available)
    Console.WriteLine($"{info.Name}: {info.Description}");

// Resolve by name
if (ReporterRegistry.TryCreate("markdown", "./output", out var reporter))
    await reporter.ReportAsync(results);
```

### Seed reporters (in NBenchmark core)

| Name       | Class              | Default output                                         |
| ---------- | ------------------ | ------------------------------------------------------ |
| `json`     | `JsonReporter`     | `benchmarks-{timestamp}-{counter}.json` in current dir |
| `markdown` | `MarkdownReporter` | `benchmark-results-{timestamp}.md` in current dir      |
| `csv`      | `CsvReporter`      | `benchmark-results.csv` in current dir                 |

The `console` reporter is provided by the optional `NBenchmark.Console` package and self-registers via `[ModuleInitializer]`.

### Registering custom reporters

```csharp
ReporterRegistry.Register("my-reporter", "Custom output format",
    outputDir => new MyReporter(outputDir));

// Then use via CLI
// dotnet run -- --reporter my-reporter
```

The factory receives the output directory when `--output` is used, or `null` when used programmatically.

## Built-in reporters

### ConsoleReporter (NBenchmark.Console package)

```xml
<PackageReference Include="NBenchmark.Console" Version="*" />
```

```csharp
using NBenchmark.Console;

// Add to host
await BenchmarkHost.Create(args)
    .AddFromAssembly<MyBenchmarks>()
    .WithReporter(new ConsoleReporter())
    .RunAsync();

// Add to suite
var results = await new BenchmarkSuite("MySuite")
    .Add("a", () => A())
    .Add("b", () => B())
    .WithReporter(new ConsoleReporter())
    .RunAsync();

// Quick mode
await Benchmark.Run(() => MyMethod()).PrintAsync();
```

Features: Spectre.Console table with rounded borders, colour-coded names (green/yellow/red by ratio), significance indicators (green checkmark ✓, grey tilde ~), ratio colours, bar chart for multi-benchmark runs, summary footer.

### JsonReporter

```csharp
new JsonReporter()           // writes to ./benchmarks-{timestamp}-{counter}.json
new JsonReporter("./output") // writes to ./output/benchmarks-{timestamp}-{counter}.json
```

Output format uses `System.Text.Json` with camelCase, indented:

```json
{
  "generatedAt": "2026-01-15T10:30:00Z",
  "results": [
    {
      "name": "MyMethod",
      "mean": 1234.5,
      "median": 1200.0,
      ...
    }
  ]
}
```

File names are timestamped with a monotonically incrementing counter, safe for multiple runs without overwriting.

### MarkdownReporter

```csharp
new MarkdownReporter()               // writes to benchmark-results-{timestamp}.md
new MarkdownReporter("results.md")   // uses the exact path provided
```

Output format:

```markdown
## Benchmark Results

_Run at 2026-01-15 10:30:00 UTC - 25 warmup / 200 measured_

| Benchmark  | Median   | Mean     |    Error |  StdDev |      P95 |      P99 |  Ratio | Sig | Alloc/op |
| ---------- | -------- | -------- | -------: | ------: | -------: | -------: | -----: | --: | -------: |
| QuickSort  | 1.24 µs  | 1.30 µs  | ±0.05 µs | 0.12 µs |  1.50 µs |  1.80 µs |  1.00x |   - |     64 B |
| BubbleSort | 15.30 ms | 15.50 ms | ±0.50 ms | 2.10 ms | 18.00 ms | 20.00 ms | 12.35x |   ✓ |    128 B |

_Error = ±95% confidence interval half-width on the mean._
```

Time values use `BenchmarkFormatter.FormatNs` (auto-scaling ns/µs/ms/s). Bytes use `FormatBytes` (B/KB/MB). Columns: Benchmark, Median, Mean, Error, StdDev, P95, P99, Ratio, Sig, Alloc/op.

### CsvReporter

```csharp
new CsvReporter()                // writes to benchmark-results.csv
new CsvReporter("./output.csv")  // writes to output.csv
```

Columns: `Name, Median, Mean, StdDev, StdErr, MarginOfError, CiLower, CiUpper, ConfidenceLevel, CoefficientOfVariation, P95, P99, Ratio, Significant, AllocPerOp`.

CSV uses double-quote escaping for names and significance values. Null/missing values are written as `null`.

## BenchmarkTable.Build

Reporters build display tables via `BenchmarkTable.Build(results)`. This is also useful for programmatic access.

```csharp
var table = BenchmarkTable.Build(results);

foreach (var row in table.Rows)
{
    Console.WriteLine($"{row.Name}: {row.Ratio:F2}x, sig={row.SignificanceLabel}");
}

// Table metadata
Console.WriteLine($"Total duration: {table.TotalDuration}");
Console.WriteLine($"Outlier mode: {table.OutlierMode}");
```

`BenchmarkTable` selects the baseline (first explicit `IsBaseline`, or the benchmark with the fastest median), computes ratios, and orders rows by median ascending.

## Output path safety

`PathValidation.ValidateOutputPath` ensures the resolved full path is under the current working directory. Throws `ArgumentException` for paths outside the working directory. All built-in reporters use this validation.

## Multiple reporters

Attach multiple reporters to the same run:

```csharp
.WithReporter(new ConsoleReporter())
.WithReporter(new MarkdownReporter("results.md"))
.WithReporter(new JsonReporter("./output"))
.WithReporter(new CsvReporter("results.csv"))
```

All reporters run sequentially in the order they were added. CLI `--reporter` flags are inserted at position 0 and run first.

## Custom reporter example

```csharp
using NBenchmark.Reporters;

public sealed class MyReporter : IReporter
{
    public string Name => "my-reporter";

    public async Task ReportAsync(
        IReadOnlyList<BenchmarkResult> results,
        CancellationToken ct = default)
    {
        var table = BenchmarkTable.Build(results);
        // Write custom output...
        foreach (var row in table.Rows.Where(r => !r.Errored))
        {
            await File.AppendAllTextAsync("output.txt",
                $"{row.Name}: {row.Median:F0} ns{Environment.NewLine}", ct);
        }
    }
}

// Register for CLI use
ReporterRegistry.Register("my-reporter", "Custom output format",
    _ => new MyReporter());
```
