---
name: nbenchmark-reporters
nbenchmarkVersion: v0.32.0
lastVerified: 2026-07-21
description: NBenchmark output and reporting. Use when the user wants console, JSON, Markdown, or CSV output, wants to control report detail levels (Simple / Standard / Advanced), stack multiple reporters, write file output to a directory, register a custom reporter via ReporterRegistry, or implement a custom IReporter or IMeasurementObserver. For running benchmarks see the core nbenchmark skill; for the --reporter/--output/--detail CLI flags see nbenchmark-host.
---

# NBenchmark Reporters

Reporters turn `IReadOnlyList<BenchmarkResult>` into output. The core `NBenchmark` package includes file reporters (JSON, Markdown, CSV) and the reporter/observer registries; the `NBenchmark.Reporters.Console` package adds the rich terminal table.

| Reporter           | Package                        | Output                           |
| ------------------ | ------------------------------ | -------------------------------- |
| `ConsoleReporter`  | `NBenchmark.Reporters.Console` | Rich terminal table (+ progress) |
| `JsonReporter`     | `NBenchmark` (core)            | JSON file per run                |
| `MarkdownReporter` | `NBenchmark` (core)            | Markdown table file              |
| `CsvReporter`      | `NBenchmark` (core)            | CSV file                         |

## When to use this skill

- Print results to the console
- Write JSON/Markdown/CSV files
- Stack multiple reporters in one run
- Choose between Simple, Standard, and Advanced detail
- Write a custom reporter and expose it via `--reporter`
- Subscribe to per-sample/per-phase telemetry via `IMeasurementObserver`

## Attaching reporters

All three modes accept `IReporter` instances. Reporters are **stackable** — call `WithReporter` more than once.

```csharp
using NBenchmark;
using NBenchmark.Reporters;          // file reporters + IReporter
using NBenchmark.Reporters.Console;  // ConsoleReporter

await new BenchmarkSuite("Demo")
    .Add("A", () => DoA())
    .Add("B", () => DoB())
    .WithReporter(new ConsoleReporter())
    .WithReporter(new MarkdownReporter("results/"))
    .WithReporter(new JsonReporter("results/"))
    .RunAsync();
```

In Harness mode, attach via code or the `--reporter` CLI flag (`json`, `markdown`, `csv`, `console`). See the `nbenchmark-host` skill / its CLI reference.

## Console output (single result)

```csharp
result.Print();              // plain-text summary, core package (optional ReportDetail arg)
await result.PrintAsync();   // rich table, requires NBenchmark.Reporters.Console
```

`new ConsoleBenchmarkProgress()` (console package, `IBenchmarkProgress`) renders a live progress bar when passed to `WithProgress(...)`. With auto-resolved counts the bar tracks the `MaxSamples` ceiling; pin `WithWarmup`/`WithIterations` for an exact total.

## File reporters

All file reporters share the constructor shape `(string outputDirectory = ".", string? name = null, ReportDetail detail = ReportDetail.Simple)`:

```csharp
new JsonReporter("results/");
new MarkdownReporter("results/", "comparison.md");
new CsvReporter("results/", detail: ReportDetail.Advanced);
```

- `outputDirectory` **must be under the current working directory** (validated; a path outside CWD throws). It is created automatically.
- When `name` is null, a timestamped name with a 3-digit counter is generated:
  - Markdown / CSV: `benchmark-results-{yyyyMMdd-HHmmss}-{NNN}.{md|csv}`
  - JSON: `benchmarks-{yyyyMMdd-HHmmss}-{NNN}.json` (note the different prefix)

### Single-result extension methods

On any `BenchmarkResult` (core package). Note these take a directory and an optional file name, not a single file path:

```csharp
await result.ToMarkdownAsync("results/");              // (outputDir = ".", string? fileName = null)
await result.ToJsonAsync("results/", "run.json");
await result.ToCsvAsync("results/");
```

### JSON envelope

`JsonReporter` writes indented camelCase JSON with enums as camelCase strings, wrapped in an envelope. JSON **always** contains the full record regardless of detail level:

```json
{
  "generatedAt": "2026-06-12T15:00:00+00:00",
  "detail": "simple",
  "results": [ { "name": "QuickSort", "median": 1234.5, "significanceVerdict": "significant", ... } ]
}
```

Each result also carries an `autoTune` object (resolved warmup/samples/ops, `warmupStop`/`sampleStop` reasons, achieved CI half-width, tuning time), or `null` for dry-run and errored results.

## Detail levels

`ReportDetail` is `{ Simple, Standard, Advanced }`. Simple is the default. Set it per reporter (`detail:` ctor arg), per suite (`.WithDetail(...)`), per host (`.WithDetail(...)`), or via `--detail simple|standard|advanced`.

### Simple — 10-column table

| Column    | Meaning                                       |
| --------- | --------------------------------------------- |
| Benchmark | Name                                          |
| Median    | Median timing (primary metric)                |
| Mean      | Arithmetic mean                               |
| Error     | ±Margin of error on the mean (with % of mean) |
| StdDev    | Sample standard deviation                     |
| P95       | 95th percentile                               |
| P99       | 99th percentile                               |
| Ratio     | Speed relative to baseline                    |
| Sig       | `✓` significant, `✗` not significant, `-` n/a |
| Alloc/op  | Mean bytes/op, or `-` if not measured         |

### Standard — same table + percentile columns and effect size

Adds the configured percentile columns (default P50/P95/P99/P99.9/Max beyond the table), `EffectMetric`, `EffectValue`, `Magnitude` (Cliff's δ magnitude label), `MarginPercent`, `OutliersRemoved`.

### Advanced — same table + per-benchmark stats block

Adds: range (Min–Max), quartiles (Q1/Q3/IQR), fences (IqrFence only), pre/post-trim sample counts + warmup, full CI bounds + margin %, CV %, skewness, kurtosis, MAD, N, (when measured) allocation median/P95/max, plus an `auto-tuned: …` line summarising the adaptive loop's decisions (resolved samples × ops, warmup length, achieved CI half-width).

### Per-reporter behaviour

| Reporter | Simple               | Standard                                   | Advanced                                                   |
| -------- | -------------------- | ------------------------------------------ | ---------------------------------------------------------- |
| Console  | Table only           | Table + standard columns                   | Table + stats block + `auto-tuned:` line below each row    |
| Markdown | Table only           | Table + standard columns                   | Table + details section (incl. `auto-tuned:` line)         |
| CSV      | Base columns         | + percentiles, effect, CI, margin %       | + quartiles, fences, shape, adaptive stats, diagnostics    |
| JSON     | Full record (always) | Full record (always)                       | Full record (always)                                       |

## Custom reporters

Implement `IReporter` from `NBenchmark.Reporters`. The interface has **three** members — note `Detail` (a settable property the host/suite assigns based on `--detail` / `WithDetail`):

```csharp
using NBenchmark;
using NBenchmark.Reporters;

public sealed class MyReporter : IReporter
{
    public string Name => "my-reporter";
    public ReportDetail Detail { get; set; } = ReportDetail.Simple;

    public async Task ReportAsync(
        IReadOnlyList<BenchmarkResult> results,
        CancellationToken cancellationToken = default)
    {
        foreach (var r in results.Where(r => !r.Errored))
            Console.WriteLine($"{r.Name}: median={r.Median:F0}ns");
    }
}
```

Attach with `.WithReporter(new MyReporter())`.

### Build comparison tables with `BenchmarkTable`

For ratio/significance tables, use `BenchmarkTable.Build(results)` instead of reimplementing the logic:

- Baseline selection — first `[Baseline]`, else fastest median
- `row.Ratio` — `result.Median / baseline.Median` (`NaN` for errored / single-benchmark runs)
- `row.SignificanceLabel` — `"✓"`, `"✗"`, or `""`
- Rows sorted by median ascending
- Run metadata: `table.RunAtUtc`, `WarmupIterations`, `MeasuredIterations`, `ConfidenceLevel`, `OutlierMode`, `TotalDuration`
- `row.AutoTune` — the per-benchmark adaptive-loop diagnostic (`AutoTuneDiagnostic?`; `null` on dry-run/errored)

```csharp
public async Task ReportAsync(IReadOnlyList<BenchmarkResult> results, CancellationToken ct = default)
{
    var table = BenchmarkTable.Build(results);
    foreach (var row in table.Rows)
    {
        if (row.Errored) { Console.WriteLine($"{row.Name}: ERROR - {row.ErrorMessage}"); continue; }
        var sig = row.SignificanceLabel is "" ? "" : $" {row.SignificanceLabel}";
        Console.WriteLine($"{row.Name}{sig}: {row.Median:F0} ns  ratio={row.Ratio:F2}x");
    }
}
```

### Register for the `--reporter` CLI flag

Register with the global `ReporterRegistry` (namespace `NBenchmark.Reporters`). The factory takes **two** arguments — the output directory and the detail level:

```csharp
using System.Runtime.CompilerServices;
using NBenchmark.Reporters;

internal static class MyReporterRegistration
{
    [ModuleInitializer]
    internal static void Register() =>
        ReporterRegistry.Register(
            "my-reporter",
            "Custom output",
            (dir, detail) => new MyReporter { Detail = detail });
}
```

Registry API:

| Member              | Signature                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `Register`          | `Register(string name, string description, Func<string?, ReportDetail, IReporter> factory)` |
| `RegisterAutoAttach`| `RegisterAutoAttach(string name, string description, Func<string?, ReportDetail, IReporter> factory)` — reporter is auto-attached to every run even when `--reporter` is not specified |
| `TryCreate`         | `TryCreate(string name, string? outputDir, ReportDetail detail, out IReporter? reporter)`   |
| `Available`         | `IReadOnlyList<ReporterInfo>` — registered factories (`ReporterInfo(string Name, string Description)`) |
| `AutoAttached`      | `IReadOnlyList<ReporterInfo>` — factories registered via `RegisterAutoAttach`                |

`Register` throws `InvalidOperationException` if the name is already taken (case-insensitive). Packages that reference `NBenchmark.*` are auto-loaded so their `[ModuleInitializer]` registrations run; the console package self-registers `console` this way. Seed reporter descriptions: `"JSON file output (one file per run)"`, `"Markdown table output"`, `"CSV file output"`.

## Observers (telemetry)

For non-perturbing per-sample / per-phase telemetry (live trace events, histograms, latency feeds), implement `IMeasurementObserver` (namespace `NBenchmark`). The interface extends `IDisposable` and exposes `OnPhase`, `OnSample`, `OnDetector`, `OnResult` callbacks receiving readonly-struct event records (`MeasurementPhaseEvent`, `SampleEvent`, `DetectorStateEvent`). Built-in implementations: `NullMeasurementObserver` (no-op singleton), `ChannelMeasurementObserver` (bounded `Channel<MeasurementEvent>` with `DropOldest` backpressure), `CompositeMeasurementObserver` (fans out to a list of children with per-dispatch isolation).

Attach with `.WithObserver(...)` (suite and host) or `BenchmarkHarness.WithObserver`. Register named factories with `ObserverRegistry` (namespace `NBenchmark.Observers` - same API shape as `ReporterRegistry`: `Register`, `RegisterAutoAttach`, `TryCreate`, `Available`, `AutoAttached`, `IsRegistered`). Composite observers can be built to stack multiple observers in one run.

See [references/observers.md](references/observers.md) for the full observer/progress API: the `IMeasurementObserver` contract, the `MeasurementEvent` tagged union and event records, the `MeasurementPhase` / `PhaseTransition` / `WarmupStopReason` / `SampleStopReason` enums, the built-in observers, `ObserverRegistry`, and the `IBenchmarkProgress` progress interface (distinct from observers).

## References

- [observers.md](references/observers.md) - `IMeasurementObserver`, event types, `ObserverRegistry`, `IBenchmarkProgress`

## Related skills

- **nbenchmark** — running benchmarks, `BenchmarkResult` fields
- **nbenchmark-host** — `--reporter`, `--output`, `--detail`, `--observer` CLI flags
- **nbenchmark-integration** — assertion-style reporting inside test frameworks
- **nbenchmark-troubleshooting** — output/path issues, interpreting warnings
