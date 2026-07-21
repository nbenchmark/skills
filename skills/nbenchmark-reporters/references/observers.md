# Observers & Progress Reference

NBenchmark has two parallel seams for live run telemetry:

- **`IMeasurementObserver`** - per-sample / per-phase telemetry stream (non-perturbing, value-type events). For live trace export, histogram streaming, OTLP pipelines.
- **`IBenchmarkProgress`** - lifecycle / percent-complete signals (no measurement payload). For progress bars and "running benchmark 3/10" indicators.

Both are invoked from the same emit points in `AdaptiveLoop`, but carry different contracts. This reference covers both.

## `IMeasurementObserver`

The observer interface. Extends `IDisposable` (the default `Dispose` is a no-op, so implementations with no unmanaged resources need no code).

```csharp
namespace NBenchmark;

public interface IMeasurementObserver : IDisposable
{
    string? Name => null;                          // default - set by the registry on resolved instances
    void IDisposable.Dispose() { }                 // default - no-op; override to release resources
    void OnPhase(in MeasurementPhaseEvent e);       // phase transition
    void OnSample(in SampleEvent e);               // one timed sample (outside the timed region)
    void OnDetector(in DetectorStateEvent e);       // snapshot of the running detector state
    void OnResult(BenchmarkResult result);          // post-trim summary for one benchmark
}
```

### Timing contract

Implementations MUST return immediately and MUST NOT block, allocate on the hot path, or do I/O. The emit points are inside the measurement loop, so a slow observer perturbs every sample. The built-in `ChannelMeasurementObserver` is the reference for how to stay non-blocking: write a value-type event to a bounded channel and let a background pump do the real work.

`OnResult` may be called with `null` on pre-runner failure sites; handle it (the channel observer drops the event).

### `Name` and dedup

The `Name` property is `null` for a programmatically attached observer. The harness/suite set `Name` on instances resolved through `ObserverRegistry` and use it to dedup when both `--observer <name>` and an auto-attached registration produce the same observer.

## Event types

### `MeasurementEvent` - tagged union

A readonly record struct that carries one of four payloads, discriminated by `EventKind`:

| `EventKind` | Payload property | Constructor |
|---|---|---|
| `Phase` (0) | `MeasurementPhaseEvent PhaseEvent` | `new MeasurementEvent(MeasurementPhaseEvent e)` |
| `Sample` | `SampleEvent SampleEvent` | `new MeasurementEvent(SampleEvent e)` |
| `DetectorState` | `DetectorStateEvent DetectorStateEvent` | `new MeasurementEvent(DetectorStateEvent e)` |
| `Result` | `BenchmarkResult? Result` | `new MeasurementEvent(BenchmarkResult result)` |

Read `Kind` to switch on the discriminator, then access the matching payload.

### `MeasurementPhaseEvent` - phase transition

A readonly record struct carrying the phase outcome:

| Field | Type | Description |
|---|---|---|
| `BenchmarkName` | `string` | Empty for `SuiteCompleted` events |
| `Phase` | `MeasurementPhase` | `Jitter` / `Calibration` / `Warmup` / `Measurement` / `SuiteCompleted` |
| `Transition` | `PhaseTransition` | `Starting` or `Completed` |
| `JitterMetric` | `double?` | Set on jitter completion |
| `DetectorSwitched` | `bool` | Set on jitter completion (true if the jitter run swapped the outlier detector) |
| `ResolvedK` | `int?` | Set on calibration completion |
| `ResolvedWarmup` | `int?` | Set on warmup completion |
| `WarmupStop` | `WarmupStopReason?` | Set on warmup completion (`Settled` / `MaxCeiling` / `ExplicitCount` / `WallClockCap`) |
| `SampleStop` | `SampleStopReason?` | Set on measurement completion (`CiTargetMet` / `MaxCeiling` / `ExplicitCount` / `WallClockCap` / `GraceCapExhausted`) |
| `Succeeded` | `bool` | `SuiteCompleted` only (true on success, false from `finally` after a harness-level exception) |

Outcome fields are `null` on `Starting` transitions and populated for `Completed`.

### `SampleEvent` - one timed sample

A readonly record struct:

| Field | Type | Description |
|---|---|---|
| `BenchmarkName` | `string` | |
| `Ordinal` | `int` | 0-based index within its phase |
| `PerOpNs` | `double` | Per-op nanoseconds = elapsed / K |
| `K` | `int` | Ops-per-sample count in effect when this sample was timed |
| `AllocDelta` | `long` | Allocation delta in bytes per K ops (0 when allocation tracking is off) |
| `Warmup` | `bool` | `true` for calibration and warmup samples; `false` for measured samples |

### `DetectorStateEvent` - detector snapshot

A readonly record struct emitted after a detector update:

| Field | Type | Description |
|---|---|---|
| `BenchmarkName` | `string` | |
| `Phase` | `MeasurementPhase` | Usually `Measurement` |
| `SampleCount` | `int` | Samples seen so far |
| `Mean` | `double` | Running mean |
| `StdDev` | `double` | Running standard deviation |
| `CiHalfWidth` | `double` | Live convergence curve (during measurement; not meaningful during calibration) |
| `CurrentK` | `int` | Current ops-per-sample |

## Enums

### `MeasurementPhase`

| Member | Value | Description |
|---|---|---|
| `Jitter` | 0 | Pre-flight environment probe; reports a jitter metric; no body invocations |
| `Calibration` | 1 | Ops-per-sample K doubling search |
| `Warmup` | 2 | Warmup plateau; ends with `WarmupStopReason` |
| `Measurement` | 3 | Collects samples that feed statistics; ends with `SampleStopReason` |
| `SuiteCompleted` | 4 | Emitted once per `RunAsync` after the suite finishes; `BenchmarkName` is empty; carries `Succeeded` |

### `PhaseTransition`

| Member | Value | Description |
|---|---|---|
| `Starting` | 0 | Phase about to run; outcome fields are null |
| `Completed` | 1 | Phase finished; outcome fields populated |

### `WarmupStopReason`

| Member | Value |
|---|---|
| `Settled` | 0 |
| `MaxCeiling` | 1 |
| `ExplicitCount` | 2 |
| `WallClockCap` | 3 |

### `SampleStopReason`

| Member | Value |
|---|---|
| `CiTargetMet` | 0 |
| `MaxCeiling` | 1 |
| `ExplicitCount` | 2 |
| `WallClockCap` | 3 |
| `GraceCapExhausted` | 4 |

## Built-in observers

### `NullMeasurementObserver`

A singleton (namespace `NBenchmark`): `NullMeasurementObserver.Instance`. All callbacks are no-ops. The hot-path guard in `AdaptiveLoop` is `observer != NullMeasurementObserver.Instance`, so attaching this singleton is observation-free. Use it as a default or in tests.

### `ChannelMeasurementObserver`

A channel-backed observer (namespace `NBenchmark.Engine`) that writes every event to a bounded `Channel<MeasurementEvent>` with `BoundedChannelFullMode.DropOldest`. The writer side is non-blocking and allocation-free per event (the event is a value type); a slow consumer can never perturb measurement because the channel drops the oldest event when full instead of blocking.

```csharp
public sealed class ChannelMeasurementObserver : IMeasurementObserver
{
    public ChannelMeasurementObserver(int capacity = 1024);
    public ChannelReader<MeasurementEvent> Reader { get; }
    public void OnPhase(in MeasurementPhaseEvent e);
    public void OnSample(in SampleEvent e);
    public void OnDetector(in DetectorStateEvent e);
    public void OnResult(BenchmarkResult result);   // drops null results
    public void Dispose();                            // completes the writer
    public void Complete();                           // explicit completion signal
}
```

`OnResult` silently drops `null` results (a `Kind=Result` / `Result=null` frame would be ambiguous to a consumer). `Dispose` and `Complete` both call `TryComplete` on the writer; safe to call multiple times.

Typical use - a background pump drains the reader while the benchmark runs:

```csharp
var observer = new ChannelMeasurementObserver(capacity: 4096);
_ = Task.Run(async () =>
{
    await foreach (var e in observer.Reader.ReadAllAsync())
    {
        switch (e.Kind)
        {
            case MeasurementEvent.EventKind.Sample:
                HandleSample(e.SampleEvent);
                break;
            case MeasurementEvent.EventKind.Phase:
                HandlePhase(e.PhaseEvent);
                break;
        }
    }
});

await new BenchmarkSuite("Demo")
    .Add("A", () => DoA())
    .WithObserver(observer)
    .RunAsync();   // observer.Dispose() runs automatically
```

### `CompositeMeasurementObserver`

A fan-out observer (namespace `NBenchmark.Engine`) that dispatches every callback to a list of children with per-dispatch isolation. Exceptions from a child are traced via `Trace.TraceWarning` and the dispatch continues to the remaining children. The fan-out is sequential, not parallel. `Dispose` fans out to each child in a try/catch.

```csharp
public sealed class CompositeMeasurementObserver : IMeasurementObserver
{
    public CompositeMeasurementObserver(IEnumerable<IMeasurementObserver> observers);
    public IReadOnlyList<IMeasurementObserver> Observers { get; }
    // OnPhase / OnSample / OnDetector / OnResult / Dispose - all fan out
}
```

The harness/suite compose observers automatically when you attach more than one; you usually don't construct this directly.

## `ObserverRegistry`

The registry (namespace `NBenchmark.Observers`) lets external packages expose named observers that users attach via `--observer <name>` on the CLI or `.WithObserver(name)` in code. It's the observer analogue of `ReporterRegistry`.

| Member | Signature | Description |
|---|---|---|
| `Available` | `IReadOnlyList<ObserverInfo>` (static) | Explicit opt-in factories registered via `Register`; lazily loads extension assemblies on first access |
| `AutoAttached` | `IReadOnlyList<ObserverInfo>` (static) | Factories registered via `RegisterAutoAttach`; auto-attached to every run even when `--observer` is not specified |
| `Register` | `void Register(string name, string description, Func<IMeasurementObserver> factory)` | Throws `InvalidOperationException` if the name is already registered in either list (case-insensitive) |
| `RegisterAutoAttach` | `void RegisterAutoAttach(string name, string description, Func<IMeasurementObserver> factory)` | Same throw contract as `Register`; the same name cannot live in both lists |
| `TryCreate` | `bool TryCreate(string name, out IMeasurementObserver? observer)` | Searches both lists; returns `true` + a fresh instance if found |
| `IsRegistered` | `bool IsRegistered(string name)` | Checks both lists without constructing an instance |

`ObserverInfo` is a sealed record: `ObserverInfo(string Name, string Description)`.

### Registering an observer

```csharp
using System.Runtime.CompilerServices;
using NBenchmark;
using NBenchmark.Observers;

internal static class MyObserverRegistration
{
    [ModuleInitializer]
    internal static void Register() =>
        ObserverRegistry.Register(
            "my-observer",
            "Streams events to my backend",
            () => new MyObserver());
}
```

Packages that reference `NBenchmark.*` are auto-loaded so their `[ModuleInitializer]` registrations run. Use `Register` for opt-in observers (user must pass `--observer my-observer`); use `RegisterAutoAttach` for observers that should always run (e.g. the `studio` observer from `NBenchmark.Studio`).

### CLI usage

```bash
dotnet run -- --observer my-observer          # attach by name (repeatable)
dotnet run -- --observer my-observer --observer studio   # stack multiple (composed)
```

Multiple `--observer` flags are composed into a `CompositeMeasurementObserver`. Auto-attached observers are added automatically even when `--observer` is not specified.

## `IBenchmarkProgress`

The progress interface (namespace `NBenchmark`). Carries lifecycle signals only - no per-sample payload, no detector state. For progress bars and "running benchmark 3/10" indicators.

```csharp
namespace NBenchmark;

public interface IBenchmarkProgress
{
    Task OnSuiteStarting(IReadOnlyList<string> benchmarkNames, int total);
    Task OnWarmupStarting(string name, int totalWarmupIterations);
    Task OnWarmupCompleted(string name);
    Task OnBenchmarkStarting(string name, int index, int total);
    Task OnIterationCompleted(string name, int iteration, int totalIterations);
    Task OnBenchmarkCompleted(BenchmarkResult result);
    Task OnSuiteCompleted(IReadOnlyList<BenchmarkResult> results);
}
```

`OnWarmupStarting` / `OnIterationCompleted` receive a `totalIterations` / `totalWarmupIterations` that is `<= 0` when the count is auto-resolved (the loop stops on a CI target). Progress UIs should treat a non-positive total as indeterminate - no percentage or ETA, just a live sample count.

## Built-in progress implementations

### `NullBenchmarkProgress`

A singleton (namespace `NBenchmark`): `NullBenchmarkProgress.Instance`. All callbacks return `Task.CompletedTask`. The default when no progress is set.

### `DefaultConsoleProgress`

A lightweight console progress reporter (namespace `NBenchmark`) that uses ANSI escape sequences (no Spectre.Console dependency). This is the default progress when no explicit progress is set and the output is a terminal. Renders an inline progress bar with ETA when the total is known, or a bouncing indeterminate-bar segment when the count is auto-resolved.

### `ConsoleBenchmarkProgress`

The Spectre.Console-based progress reporter (namespace `NBenchmark.Reporters.Console`, in the `NBenchmark.Reporters.Console` package). Renders a richer progress bar with ANSI colours. Pass to `WithProgress(...)`:

```csharp
await BenchmarkHarness.Create(args)
    .AddFromAssembly<StringBenchmarks>()
    .WithReporter(new ConsoleReporter())
    .WithProgress(new ConsoleBenchmarkProgress())
    .RunAsync();
```

Both console progress implementations print a one-line summary per benchmark on completion: `✓ Name  1.23 µs  · 12/3/0 GC  (1.4s)` (or `✗` with the error message on failure).

## Attaching observers and progress

| Mode | Observer | Progress |
|---|---|---|
| Single | (not supported - use Suite) | (not supported - use Suite) |
| Suite | `.WithObserver(observer)` | `.WithProgress(progress)` |
| Harness | `.WithObserver(observer)` or `--observer <name>` | `.WithProgress(progress)` |

Observers and progress are invoked from the same emit points in `AdaptiveLoop`, so both fire on every phase transition, sample, and benchmark completion. The observer carries the measurement payload; the progress carries the lifecycle signal. Attach both when you want both a live UI and a telemetry stream.

## Isolated-process note

Programmatic observers attached via `.WithObserver(instance)` do NOT cross the process boundary - an isolated child runs with `NullMeasurementObserver` only. Named observers resolved through `ObserverRegistry` (via `--observer <name>` or auto-attach) DO cross: the harness re-registers them in the child process. If you need telemetry from an isolated benchmark, register your observer through `ObserverRegistry` and attach by name rather than passing an instance directly.

## Related references

- The `nbenchmark-reporters` skill - reporter pipeline (distinct from observers)
- [cli.md](../nbenchmark-host/references/cli.md) - the `--observer` flag
- The `nbenchmark-troubleshooting` skill - diagnosing noisy results (observers are non-perturbing by design)