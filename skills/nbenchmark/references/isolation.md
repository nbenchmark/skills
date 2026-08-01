# Process Isolation & Workers Reference

Every NBenchmark mode - Single, Suite, Harness, and the test-framework integrations - measures in a **dedicated worker process by default**. This reference covers the worker model, the capture fallback, the prepared-state remedy, runtime profiles, and the CLI flags that enforce or diagnose isolation.

## Why a worker

JIT tiering, dynamic PGO, and GC flavour are fixed at process start. A benchmark measured in a host process inherits whatever the preceding benchmarks left behind - a tier-1 compilation, a warmed cache, a server GC - and the runtime exposes no managed read-back for tiering, so a process cannot even report its own JIT configuration. The only honest answer is to measure in a freshly spawned process with a known startup configuration.

The process boundary is the delivery mechanism; the runtime profile (below) is the payload.

## The worker model

| Mode | Default | Opt out |
|---|---|---|
| Single | One worker per `Benchmark.Run` call | `Benchmark.RunInProcess` |
| Suite | One worker for the whole suite (paired comparisons stay within-process) | `.WithIsolation(false)` |
| Harness | One worker per benchmark class | `--in-process`, `[InProcess]`, `.WithIsolation(false)` |
| Test integration | One worker per test method | `[AllowInProcessGate]` (waives the isolation requirement) |

The worker is a dedicated `nbworker` executable deployed beside the assembly under test. The host launches it, frames the work over a binary protocol, and reads results back - never via stdout, so worker console output cannot corrupt the data. Workers self-exit if the coordinator dies (no supervisor), and a wedged worker is killed after a wall-clock ceiling derived from the tuning budget.

> `WithIsolation()` (no argument) still works but is **superseded** - isolation is the default. `WithIsolation(false)` opts a suite back into the host process, deliberately and silently (no warning).

### `Benchmark.Warmup()`

```csharp
Benchmark.Warmup();   // pre-warm a worker in the background
```

Optional. A worker costs roughly 70 ms to start; this hides that cost from the first `Benchmark.Run`. Worth calling at the start of a script, or before a timed section of a tool. Fire-and-forget by design - the run that needs the worker will report any failure better than this can.

## The capture fallback

A benchmark body that captures a local variable, parameter, or `this` from its enclosing scope cannot be addressed in a worker. Captured values live only in the process that created them, and reconstructing them was measured to return plausible but silently wrong numbers rather than failing. Isolation is therefore **refused** and the body falls back to the host with a labelled reason.

```csharp
var data = BuildData();   // a local in the caller's scope

// Captures `data` -> runs in-process (InProcessCapturedState)
Benchmark.Run(() => Sort(data));

// Prepared state -> runs in a worker
Benchmark.Run(prepare: () => BuildData(), body: d => Sort(d));
```

The result's `IsolationStatus` field names exactly why a measurement ran where it did (see below). The console/reporter shows an `Iso` column in mixed-isolation tables, and the ratio column is withheld as `n/a` between rows measured under different runtime configurations - the dominant term across a process boundary is the runtime config, not the code.

### Suite-mode capture

In Suite mode, **one capturing body takes every sibling in-process**. The suite is a single unit of paired comparison, and a paired ratio must be a within-process statement. A capture in one `Add` body therefore degrades the whole suite, not just that row.

### What the analyzer reports

`NBenchmark.Analyzers` ships **NB0014** (Info severity): "Benchmark body captures state and cannot be isolated." It fires for capturing lambdas passed to both `Benchmark.Run*` and `BenchmarkSuite.Add`. Raise it to warning severity in `.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.NB0014.severity = warning
```

See the `nbenchmark-troubleshooting` skill for the full analyzer table.

## `IsolationStatus`

The `IsolationStatus` enum (on `BenchmarkResult`) is the single source of truth for where a measurement ran and why. The `RuntimeProfileName` field says *what* configuration was used; `IsolationStatus` says *why* it was that one.

| Value | Label | When |
|---|---|---|
| `Isolated` | `isolated` | Measured in a dedicated worker launched with the requested runtime profile. The only status under which the reported runtime configuration was chosen rather than inherited. |
| `InProcessRequested` | `in-process` | Measured in the host because that is what was asked for - `--in-process`, `WithIsolation(false)`, `[InProcess]`, `--dry-run`, or `Benchmark.RunInProcess`. Nothing was refused. |
| `InProcessCapturedState` | `in-process (captures)` | The body captures state from its enclosing scope. Remedy: pass the preparation as its own delegate (prepared state, below). |
| `InProcessLiveFixture` | `in-process (fixture)` | Instances come from live code in this process - an instance factory, a service provider, or a test fixture. A worker can construct a type but cannot reproduce a factory it has never seen. Remedy: supply a static factory. |
| `InProcessUnaddressablePlan` | `in-process (inline)` | The suite is built inline and has no addressable entry point for a worker. Remedy: add a `[BenchmarkPlan]` factory. |
| `InProcessNoWorker` | `in-process (no worker)` | No measurement worker was available - usually an incomplete package restore or `NBenchmarkDeployWorker=false`. This is a deployment problem, not a property of the benchmark. |

Helpers: `status.IsIsolated()` (true only for `Isolated`), `status.ToLabel()` (short column label), `status.ToRemedy()` (one-line fix for the table footer, or `null`).

## Prepared state

The remedy for the capture fallback. Hand the worker a **recipe** for the state rather than a value it cannot have. The `prepare` delegate runs once per benchmark, before warmup, in the process that measures - so the cost of building the state is never inside a reading.

### Single mode

```csharp
// prepare builds the state; body receives what prepare returned
BenchmarkResult r = Benchmark.Run(
    prepare: () => BuildData(),
    body:    d  => Sort(d));
```

Overloads exist for sync/async, void/returning, and `RunRaw` variants - the same shape as the no-state `Run` overloads, with `Func<TState> prepare` prepended:

| Method | Body shape |
|---|---|
| `Run<TState>(Func<TState>, Action<TState>, ...)` | sync void |
| `Run<TState, T>(Func<TState>, Func<TState, T>, ...)` | sync returning |
| `RunAsync<TState>(Func<TState>, Func<TState, Task>, ...)` | async void |
| `RunAsync<TState, T>(Func<TState>, Func<TState, Task<T>>, ...)` | async returning |
| `RunRaw<TState>` / `RunRawAsync<TState>` | raw-sample variants |

Both `prepare` and `body` must capture nothing themselves. `prepare` runs **once**, not per iteration - a body that mutates its state sees the mutation on every iteration after the first (`d => Array.Sort(d)` sorts an already-sorted array from the second sample onward). Where that matters, reset via the per-iteration hooks on `BenchmarkSuite`, which run outside the timed region.

### Suite mode

```csharp
var results = await new BenchmarkSuite("Sorting")
    .WithState(() => BuildData())           // prepare, runs once per benchmark in the worker
    .Add("BubbleSort", d => BubbleSort(d))
    .Add("QuickSort",  d => QuickSort(d))
    .WithBaseline("QuickSort")
    .RunAsync();
```

`WithState<TState>(Func<TState> prepare)` returns a `BenchmarkSuite<TState>` whose `Add` overloads take the state-typed body. The state is built fresh in the worker for each benchmark in the suite.

> A suite carrying both `WithState` and `WithParameter` cannot isolate - the parameter sweep and the state factory are not composable across a process boundary.

## `[BenchmarkPlan]` - the escape hatch

For suites holding live state a worker must build itself - setup/teardown, captured locals, custom detectors, instance factories - use a `[BenchmarkPlan]` factory. The worker invokes the factory directly in its own process, so nothing needs to be serializable.

```csharp
using NBenchmark.Attributes;

await BenchmarkSuite.RunPlansAsync<Plans>();

static class Plans
{
    [BenchmarkPlan]
    public static BenchmarkSuite Serialization() =>
        new BenchmarkSuite("serialization")
            .Add("json", () => SerializeJson())
            .Add("msgpack", () => SerializeMsgPack())
            .WithBaseline("json");
}
```

| Member | Purpose |
|---|---|
| `[BenchmarkPlan]` | Marks a **static, parameterless** method returning a `BenchmarkSuite` as a plan. The method must not capture anything. `AttributeUsage = Method`. Optional `Name` property. |
| `BenchmarkSuite.RunPlanAsync(factory, ct)` | Run one plan: pass the method group directly (the attribute is not required here - the method group is itself the address). |
| `BenchmarkSuite.RunPlansAsync<T>(ct)` | Run every `[BenchmarkPlan]` method on `T`. |
| `BenchmarkSuite.RunPlansAsync(typeof(Plans), ct)` | Same, non-generic. |

A wrongly-shaped `[BenchmarkPlan]` method throws rather than skipping. **Suite-mode multi-runtime runs require `[BenchmarkPlan]** - inline suites use metadata tokens that are only valid within the build that produced them, and a factory is resolved by name, which is stable across builds.

## `WithRequireIsolation()` (Suite mode)

```csharp
await new BenchmarkSuite("Demo")
    .Add("a", () => DoA())
    .WithRequireIsolation()      // fail if any body cannot be isolated
    .RunAsync();
```

Fails the run when a body falls back to in-process for any reason other than an explicit opt-out. This is the suite-mode analogue of the harness `--strict-isolation` flag. There is no harness `WithRequireIsolation`; use `--strict-isolation` on the CLI.

## Runtime profiles

A `RuntimeProfile` is the runtime-startup configuration a benchmark is measured under: JIT tiering, dynamic PGO, ReadyToRun, and GC flavour. These knobs are the first-class concept because none of them can be changed in an already-running process - the runtime reads them once at startup. That, and not cross-benchmark contamination, is the real reason a measurement needs its own process.

| Profile | `Name` | Tiering | PGO | R2R | GC | Use for |
|---|---|---|---|---|---|---|
| `RuntimeProfile.SteadyState` | `steady-state` | off | off | off | workstation | **Default.** Fully-optimized steady-state throughput. Removes the dominant source of measurement error for short bodies. Wrong for cold-start/first-call. |
| `RuntimeProfile.Production` | `production` | on | on | on | workstation | "What will my users see?" Imprecise (3.27x spread measured) - raise the launch count and read the cross-launch interval. |
| `RuntimeProfile.ServerGc` | `server-gc` | off | off | off | server, non-concurrent | `SteadyState` plus server GC, for code that will run under a server-GC host (ASP.NET Core). Allocation-heavy benchmarks behave differently here. |
| `RuntimeProfile.Host` | `host` | (inherit) | (inherit) | (inherit) | (inherit) | Inherit whatever the host process started with, set nothing. The only honest profile for an in-process measurement, and for reproducing an older result. |

Apply a profile via `.WithRuntimeProfile(profile)` (suite and harness) or `--runtime-profile <name>` (CLI). The profile is applied to workers through their environment block at launch:

| Knob | Environment variable |
|---|---|
| Tiering | `TieredCompilation` |
| PGO | `TieredPgo` |
| ReadyToRun | `ReadyToRun` |
| Server GC | `gcServer`, `gcConcurrent` |

An in-process benchmark inherits the host's configuration and reports `RuntimeProfileName = "host"` - the profile cannot be applied after startup. Custom profiles can be built with `ExtraEnvironment` entries; results under different profiles are **never compared** (ratio/significance/threshold checks partition by runtime moniker and profile).

### Result fields

| Field | Type | Description |
|---|---|---|
| `RuntimeProfileName` | `string` | The profile name (`"steady-state"`, `"production"`, `"server-gc"`, `"host"`, or a custom name). Default `"host"`. |
| `RuntimeKnobs` | `string` | A compact summary of the applied knobs, surfaced in reporter headers. |

The console and Markdown reporters print a header line such as `> Runtime: **steady-state** (tiered=off pgo=off r2r=off)`. Mixed-profile tables are explicitly warned as non-comparable.

### Suppressing the runtime-profile warning

```sh
NBENCHMARK_SUPPRESS_RUNTIME_PROFILE_WARNING=1
```

Suppresses the guidance that prints when an in-process measurement reports `host` without an explicit profile. The profile itself is unchanged.

## Worker deployment

The `nbworker` executable is deployed beside the assembly under test by the NBenchmark MSBuild targets. The global tool launches the worker deployed beside the *assembly under test*, not the one beside the tool - so a framework-dependent net8.0 build is measured by the net8.0 worker.

| Control | Effect |
|---|---|
| `NBenchmarkDeployWorker=false` | MSBuild property to skip worker deployment. |
| `NBENCHMARK_WORKER_PATH=<path>` | Environment variable to override worker discovery. |

If every result reports `InProcessNoWorker` (`in-process (no worker)`), the target project was built without the worker - check it references `NBenchmark` and has not set `NBenchmarkDeployWorker=false`.

## CLI flags (Harness mode)

| Flag | Effect |
|---|---|
| `--in-process` | Run every benchmark in the host process - the opt-out from the default. `[InProcess]` on a method, or `WithIsolation(false)`, do the same in code. `--dry-run` is always in-process. |
| `--strict-isolation` | Fail the run (exit code 1) if any benchmark was not isolated. Failures are grouped by cause with remedies. The advisory warning nobody reads is indistinguishable from none; this makes it enforceable. |
| `--verify-isolation` | Re-measure in-process and print how much isolation changed each benchmark. Publishes nothing. Skipped under `--runtimes` (this process is one runtime). |
| `--runtime-profile <name>` | `steady-state` (default), `production`, `server-gc`, or `host`. Applied to workers; in-process benchmarks inherit the host. |
| `--stream-samples` | Forward the live per-sample observer stream across the worker boundary (batches of 128 samples or 100 ms, whichever comes first). Withdrawn if no observer is attached. |
| `--emit-raw` | Lift the 4096-sample cap on raw samples returned from a worker. Programmatic equivalents: `MeasurementOptions.MaxRawSamples`, `MeasurementOptions.UnboundedRawSamples`. |
| `--runtimes <list>` | Cross-runtime comparison. Each runtime builds and runs in its own worker. `--runtimes` overrides `--in-process`. Suite-mode multi-runtime requires `[BenchmarkPlan]`. |
| `--launch-count <n>` | Replicate count - separate worker processes. Harness default 3; Single/Suite default 1. `WithLaunchCount(n)` in code. |

See the `nbenchmark-host` CLI reference for the full flag list.

## Related references

- [measurement-options.md](measurement-options.md) - `MaxRawSamples`, `StreamSamples`, and the `LaunchCounts` static class
- [benchmark-result.md](benchmark-result.md) - `IsolationStatus`, `RuntimeProfileName`, `RuntimeKnobs` fields
- The `nbenchmark-host` skill / its CLI reference - `--strict-isolation`, `--verify-isolation`, `--runtime-profile`
- The `nbenchmark-integration` skill - `RequireIsolation`, `[AllowInProcessGate]`, `TestBodyIsolation`
- The `nbenchmark-troubleshooting` skill - NB0014, the capture fallback, worker deployment