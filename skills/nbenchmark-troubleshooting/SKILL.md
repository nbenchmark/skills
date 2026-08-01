---
name: nbenchmark-troubleshooting
nbenchmarkVersion: v0.37.0
lastVerified: 2026-08-01
description: Diagnose and fix NBenchmark problems. Use when benchmarks report 0 ns, results are noisy or have a large Error/wide confidence interval, the Sig column is blank, benchmarks aren't discovered, a class can't be instantiated, a benchmark fell back to in-process (capture fallback, missing worker), or the NBenchmark.Analyzers diagnostics NB0001-NB0014 fire. Also covers tuning iterations/warmup/outlier mode and PerClass instance-lifetime contamination. For normal usage see the core nbenchmark, nbenchmark-host, and nbenchmark-reporters skills.
---

# NBenchmark Troubleshooting

Symptom → cause → fix for common measurement problems, plus the full analyzer reference.

## When to use this skill

- A benchmark reports `0 ns`
- Results are noisy / Error column is large / CI is wide
- The `Sig` column is blank
- A `[Benchmark]` method isn't discovered
- "Could not instantiate" errors
- A benchmark ran in-process when you expected a worker (`Iso` column shows `in-process (captures)` / `in-process (no worker)`)
- An NB0001-NB0014 analyzer diagnostic appears
- `[InstanceLifetime(PerClass)]` state-contamination warnings (NB0011 / NB0013)
- Choosing iterations, warmup, or outlier mode

## Analyzers (NB0001-NB0014)

Install `NBenchmark.Analyzers` for compile-time diagnostics (ships analyzers + code fixes; runs automatically in the IDE and `dotnet build`). Categories: `NBenchmark.Usage`, `NBenchmark.Performance`, `NBenchmark.Configuration`.

| ID     | Title                                                      | Severity | Fix                                                   |
| ------ | ---------------------------------------------------------- | -------- | ----------------------------------------------------- |
| NB0001 | Benchmark class needs a public parameterless constructor   | Warning  | Add one, or use `NBenchmark.DependencyInjection`      |
| NB0002 | `[Benchmark]` method must not be `static`                  | Error    | Remove `static` (has a code fix)                      |
| NB0003 | `[BenchmarkCase]` / `[BenchmarkCases]` must match method parameters | Error | Align argument count/types with parameters          |
| NB0004 | `[Benchmark]` body has no observable side effects          | Error    | Return a value or add a side effect                   |
| NB0005 | `[Benchmark]` body does no observable work (empty)        | Error    | Put real work in the body                             |
| NB0006 | Multiple `[Benchmark(Baseline = true)]` in one class       | Error    | Keep exactly one baseline                              |
| NB0007 | Duplicate lifecycle method                                 | Error    | One method per lifecycle attribute                    |
| NB0008 | `[Benchmark]` `Iterations`/`WarmupIterations` out of range | Error    | 0-100,000 / 0-10,000 (or `-1` for default)            |
| NB0009 | `MeasurementOptions` value out of range                    | Error    | Fix `Iterations`/`WarmupIterations`/`OpsPerSample`/`ConfidenceLevel` |
| NB0010 | Throwaway lambda body (`Action` overloads)                 | Warning  | Return a value or add a side effect                   |
| NB0011 | `[InstanceLifetime(PerClass)]` with a scoped service       | Warning  | Switch to `PerMethod` (code fix) or implement `IStateReset` (code fix) |
| NB0012 | `[BenchmarkCases] cannot be combined with `[BenchmarkCase]` | Error  | Use one or the other                                   |
| NB0013 | `[InstanceLifetime(PerClass)]` with a mutable instance field accessed by multiple `[Benchmark]` methods | Warning | Switch to `PerMethod` or make the field readonly |
| NB0014 | Benchmark body captures state and cannot be isolated        | Info     | Pass the prepared state as its own delegate: `Benchmark.Run(prepare:, body:)` / `.WithState(...)`, or `[BenchmarkPlan]` for a whole suite |

`Error` severity blocks the build. `Warning` (NB0001, NB0010, NB0011, NB0013) is reported but does not block. **Info** (NB0014) is reported but does not block; raise it with `dotnet_diagnostic.NB0014.severity = warning` in `.editorconfig` when you want it visible. NB0004 is conservative (may have false positives - suppress if the analyzer can't see the work).

NB0010 inspects **only** the `Action` (void) overloads of `Benchmark.Run` / `Benchmark.RunAsync` / `Benchmark.RunRaw` / `Benchmark.RunRawAsync`; value-returning and async overloads (`Run<T>`, `RunAsync`, `RunAsync<T>`, `RunRaw<T>`, …) are not flagged.

NB0014 fires for capturing lambdas passed to both `Benchmark.Run*` and `BenchmarkSuite.Add`. In Suite mode, a capture in one body takes every sibling in-process, so the message names that. Parameterized `Add` bodies are not reported (parameters travel as serialized constants, so sweeps still isolate). `setup:` / `teardown:` delegates on `Add` are not measured bodies and are not reported.

### Code fixes

Two code fixes ship with `NBenchmark.CodeFixes`:
- **NB0002** — "Remove static modifier"
- **NB0011** — "Use `InstanceLifetime.PerMethod`" or "Implement `IStateReset`"

### Suppressing a rule

```csharp
#pragma warning disable NB0004
// ...benchmark...
#pragma warning restore NB0004
```

```ini
# .editorconfig
[*.cs]
dotnet_diagnostic.NB0004.severity = none
```

## Zero or unexpected results

| Symptom                    | Cause                                                                                  | Fix                                                                        |
| -------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Result shows `0 ns`        | **Dead-code elimination** — the body has no observable side effect                     | Use a value-returning overload (`Func<T>`) or add a side effect; see below |
| All results zeroed         | Dry-run active (`--dry-run`, or `Iterations = 0`)                                       | Remove `--dry-run`; leave `Iterations` unset (auto) or set `Iterations > 0`             |
| `MarginOfError` is `±0 ns` | Pinned to a single sample (`n < 2`), or all samples identical (timer coarser than the operation) | Leave the sample count auto; let ops-per-sample auto-calibrate (or pin `--ops-per-sample`) so a batch beats timer resolution |
| `Sig` column blank / `-`   | Fewer than 2 samples per group for the significance test                               | Don't pin `--iterations`/`--min-samples` below 2; ensure ≥2 non-errored benchmarks      |

### Preventing dead-code elimination

```csharp
// BAD — no observable effect, may report 0 ns
Benchmark.Run(() => { var x = 1 + 2; });

// GOOD — return value is consumed by the runner's sink
Benchmark.Run(() => Compute());
Benchmark.Run(() => int.Parse("12345"));
```

## Measurement variability

| Symptom                       | Likely cause                                                                     | Fix                                                                                                          |
| ----------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Large Error / wide CI         | CI target not tight enough, or sampling hit its ceiling                          | Lower `--ci-target` (e.g. `0.01`), raise `--max-samples`, or `--auto-tune thorough`                          |
| Large Error / wide CI         | OS scheduling noise                                                              | Keep/confirm `OutlierMode.IqrFence` (the default)                                                            |
| Large Error / wide CI         | Laptop thermal throttling                                                        | Raise `--min-warmup`, run plugged in, cap the run with `--max-tuning-time`                                   |
| High StdDev                   | GC / allocation noise                                                            | `.WithAllocations()` to diagnose; for GC-focused benchmarks, consider disabling `ForceGcBeforeEachIteration` |
| Bimodal warning in `Warnings` | Trimmed slow samples form a tight secondary cluster (a second execution profile) | Investigate a cold/warm path split; isolate with `[IsolatedProcess]` (Harness mode) or stabilize inputs         |

### Outlier mode quick reference

| Mode                         | When to use                                                |
| ---------------------------- | ---------------------------------------------------------- |
| `IqrFence` (default)         | General purpose; the IQR fence adapts to the data's spread |
| `RemoveTop5Percent`          | Fixed quota — always drop the slowest 5%                   |
| `RemoveTopAndBottom5Percent` | When fast outliers (e.g. cache hits) also skew results     |
| `MedianAbsoluteDeviation`   | Robust to extreme outliers; uses the scaled MAD as a fence |
| `None`                       | Latency-tail analysis where every sample matters           |

## Discovery & setup errors

| Symptom                                       | Cause                                                                     | Fix                                                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `[Benchmark]` method not discovered           | Method is `static`, class is `abstract`, or the assembly isn't registered | Run `--list`; make the class public/non-abstract and the method an instance method; `AddFromAssembly` it |
| "Could not instantiate MyClass"               | No public parameterless constructor                                       | Add one, use `[BenchmarkSetup]`, or add `NBenchmark.DependencyInjection`                                 |
| Benchmarks run in a different order each time | Random order is the default (reduces systematic bias)                     | `--order declaration` or `.WithRunOrder(RunOrder.Declaration)`; pin with `--seed`                        |
| `--output` rejected                           | Output directory must be under the current working directory              | Use a path inside the CWD (it's created automatically)                                                   |
| NB0011 / NB0013 on a `[Benchmark]` class      | `[InstanceLifetime(PerClass)]` sharing mutable state or scoped services across `[Benchmark]` methods | Switch to `[InstanceLifetime(PerMethod)]`, make the field readonly, or implement `IStateReset.ResetAsync` to make the reset explicit |

## Process isolation

Isolation is the default in every mode - each benchmark runs in a dedicated worker process. The `Iso` column (and `BenchmarkResult.IsolationStatus`) tells you where a measurement ran and why.

| Symptom | Cause | Fix |
|---|---|---|
| `in-process (captures)` on a result | The body captures a local/parameter/`this` from its enclosing scope; the worker cannot reconstruct captured values | Pass the prepared state as its own delegate: `Benchmark.Run(prepare:, body:)` / `.WithState(...)`. In Suite mode, one capture takes all siblings in-process - fix every capturing body |
| `in-process (fixture)` on a test | Test class uses `IClassFixture` / `ITestOutputHelper` / constructor injection, or has object-graph arguments | That test cannot run in a worker; add `[AllowInProcessGate]` if you still want a ratio gate, or restructure the test so the worker can build the state |
| `in-process (no worker)` on every result | No measurement worker was deployed - incomplete package restore or `NBenchmarkDeployWorker=false` | Check the target project references `NBenchmark` and has not set `NBenchmarkDeployWorker=false`; try a clean restore |
| `in-process (inline)` on a suite | The suite is built inline with no addressable entry point | Add a `[BenchmarkPlan]` factory; see the core skill's [isolation reference](../nbenchmark/references/isolation.md#benchmarkplan---the-escape-hatch) |
| `--strict-isolation` fails the run | One or more benchmarks fell back to in-process for a non-opt-out reason | Read the grouped failure output; apply the remedy per cause (capture -> prepared state; no worker -> deployment) |
| Ratio shows `n/a` in a table | Rows were measured under different runtime configurations (isolated vs in-process, or different runtime profiles) | Expected behaviour - a cross-process ratio would describe two processes more than two code paths. Make all rows use the same isolation/profile, or read the within-group ratios |
| `RuntimeProfileName` is `host` on a result you expected to isolate | The body fell back to the host, or you passed `--in-process` / `WithIsolation(false)` / `[InProcess]` / `--dry-run` | Check `IsolationStatus`; if `InProcessRequested`, that is the opt-out. If you want a specific profile, set `--runtime-profile` (it applies to workers only) |

## Instance lifetime & state contamination

`InstanceLifetime.PerClass` (the default) reuses one instance across all `[Benchmark]` methods in a class. That's fine for stateless benchmarks, but if the class has mutable instance fields or scoped-service constructor dependencies, the second method can observe state cached by the first - violating the significance test's independence assumption and producing misleading results.

- **NB0011** fires when a `PerClass` class injects a constructor parameter that looks like a scoped service (any reference type that isn't `string`, a well-known ambient type like `HttpContext`/`IServiceProvider`/`CancellationToken`, or a stateless generic like `ILogger<T>`/`IOptions<T>`).
- **NB0013** fires when a `PerClass` class has a mutable instance field accessed from two or more `[Benchmark]` methods.
- Both have code fixes: switch to `PerMethod`, or (for NB0011) implement `NBenchmark.Lifecycle.IStateReset` (a `ResetAsync(CancellationToken)` method the harness calls between methods) to make the reset explicit.
- A non-fatal `PerClassIndependenceWarning` is also emitted at runtime when a `PerClass` class has >1 `[Benchmark]` method. Suppress it with `MeasurementOptions.SuppressPerClassIndependenceWarning = true` when sharing is intentional.

## Statistical significance

NBenchmark runs a significance test against the baseline when `EnableSignificance = true` (default) and there are ≥2 non-errored benchmarks. The default strategy is **Mann-Whitney U** for two groups and **Kruskal-Wallis** with post-hoc pairwise tests for three or more groups.

- Needs at least **2 samples per group** — with fewer, the verdict is `NotTested` and the `Sig` column shows `-` / blank. (Earlier guidance that said 5 was incorrect.)
- For small, tie-free samples (combined n ≤ 20) an exact permutation p-value is used; otherwise a normal approximation with tie and continuity corrections.
- A result is **Significant** when its p-value < `SignificanceLevel` (default `0.05`; tune via `.WithSignificanceLevel(...)` or `--alpha`).
- The `MinimumPracticalEffect` gate (default `0.147` - a "small" Cliff's δ) downgrades a statistically-significant but practically-negligible result to `NotSignificant` and forces the magnitude label to `neg`. Set to `0` for p-value-only semantics, or `null` to disable the gate entirely.
- `Sig` symbols: `✓` significant, `✗` not significant, `-` not applicable (baseline / disabled / not tested).
- Significance ≠ importance — always read the **Ratio** column alongside it. A tiny, statistically-significant difference may not matter.

## Tuning cheatsheet

| Goal                                   | Setting                                                                     |
| -------------------------------------- | --------------------------------------------------------------------------- |
| Tighter confidence interval            | Lower `--ci-target` (e.g. `0.01`) or `--auto-tune thorough`; or pin a larger `--iterations` |
| Stable sub-microsecond timing          | Let ops-per-sample auto-calibrate (or pin `--ops-per-sample`); the batch spans ~1 µs so one timer read beats sub-100 ns resolution |
| Cold-start / JIT cost                  | `WarmupIterations = 0` (or `1`)                                             |
| Stronger evidence before "significant" | Lower `SignificanceLevel` (e.g. `0.01`) / `--alpha 0.01`                    |
| Filter out tiny-but-significant noise | Set `MinimumPracticalEffect` above the noise floor (default `0.147` = "small" Cliff's δ); set to `0` for p-value-only semantics |
| Wider/more conservative Error          | Higher `ConfidenceLevel` (e.g. `0.99`)                                      |
| Cross-launch stability                 | `WithLaunchCount(n)` (or `--launch-count n`); repeats in a fresh worker per launch and populates `LaunchStatistics` (incl. `ProcessVarianceRatio` / `BetweenLaunchDispersion`). Harness default is 3. |
| Clean-room run (no warmup carry-over)  | Isolation is the default in every mode. `[IsolatedProcess]` in Harness mode gives a benchmark its own dedicated worker (finest granularity); `Benchmark.RunInProcess` / `WithIsolation(false)` / `--in-process` are the deliberate opt-outs |
| Reproducible run order                 | `--order declaration` or `--seed <n>`                                       |
| GC isolation between iterations        | `MeasurementProfile.Independent` (or `--profile independent`) - forces GC before each iteration |
| Cross-runtime comparison               | `[Runtimes(RuntimeMoniker.Net8, RuntimeMoniker.Net10)]` on the benchmark class. Suite-mode multi-runtime requires `[BenchmarkPlan]`. |
| Steady-state throughput (default)      | `--runtime-profile steady-state` (tiering/PGO/R2R off) - the default, removes the dominant source of measurement error for short bodies |
| "What users see" measurement            | `--runtime-profile production` (tiering/PGO/R2R on) - imprecise; raise `--launch-count` and read the cross-launch interval |
| Fail CI if not isolated                 | `--strict-isolation` (fails the run if any benchmark fell back to in-process) |

## Related skills

- **nbenchmark** — measurement options, result fields, common patterns, worker isolation (`IsolationStatus`, prepared state, runtime profiles)
- **nbenchmark-host** — discovery, CLI, `[IsolatedProcess]`/`[InProcess]`, `--strict-isolation`, DI
- **nbenchmark-reporters** — output formats and detail levels, the `Iso` column
- **nbenchmark-integration** — flaky performance thresholds in CI, `RequireIsolation`, `[AllowInProcessGate]`
