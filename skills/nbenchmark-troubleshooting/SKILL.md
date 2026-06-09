---
name: nbenchmark-troubleshooting
nbenchmarkVersion: v0.1.0
lastVerified: 2026-06-09
description: Guide for diagnosing and fixing common NBenchmark issues. Use when the user gets analyzer warnings/errors, sees incorrect benchmark results (0 ns, large error, missing benchmarks), or needs to tune measurement parameters.
---

# NBenchmark Troubleshooting

## When to use this skill

User asks about:

- Compiler warnings/errors from NBenchmark analyzers (NB0001-NB0010)
- "Why does my benchmark show 0 ns?"
- "Why is the error margin so large?"
- "Why is my benchmark not discovered?"
- "Why does significance show '~' (not significant)?"
- Host mode errors ("Could not instantiate")
- Regression detection with `--threshold-pct`
- Tuning iterations, warmup, outlier mode

## Analyzer Diagnostics (NB0001-NB0010)

Install: `dotnet add package NBenchmark.Analyzers`

### NB0001 - Missing parameterless constructor (Warning)

**Cause:** A class with `[Benchmark]` methods has no public parameterless constructor. Activator.CreateInstance fails at runtime.

**Fixes:**

1. Add a public parameterless constructor
2. Use `NBenchmark.DependencyInjection` to resolve from a DI container

```csharp
// Fix: add parameterless constructor
public class MyBenchmarks
{
    public MyBenchmarks() { }

    [Benchmark]
    public void Measure() { }
}

// Or: use DI
BenchmarkHost.Create(args)
    .UseDependencyInjection<MyBenchmarks>(serviceProvider)
    .RunAsync();
```

Structs are not flagged (implicit zero-init constructor works).

### NB0002 - Static benchmark method (Error)

**Cause:** A method is marked `[Benchmark]` but is `static`. Only instance methods are discovered.

**Fix:** Remove the `static` keyword. An automatic code fix exists in `NBenchmark.CodeFixes`.

### NB0003 - BenchmarkArguments arity mismatch (Error)

**Cause:** Number of `[BenchmarkArguments]` values doesn't match the method's parameter count, or `[BenchmarkArguments]` is on a parameterless method.

**Fix:** Match argument count to parameter count, or remove `[BenchmarkArguments]`.

### NB0004 - No observable side effects (Info)

**Cause:** A void `[Benchmark]` method body has no observable side effects. The JIT may eliminate it, producing 0 ns results.

**Fix:** Return a value from the benchmark method, or add a side effect (method call, field write).

```csharp
// Bad: JIT may eliminate this
[Benchmark]
public void Measure()
{
    for (var i = 0; i < 1000; i++) { }
}

// Good: return a value
[Benchmark]
public int Measure()
{
    var sum = 0;
    for (var i = 0; i < 1000; i++) sum += i;
    return sum;
}
```

The heuristic checks for: method calls, field/property writes, ref/out args, return values, await expressions, and allocations. May have false positives since it is conservative.

### NB0005 - Empty benchmark body (Warning)

**Cause:** A void `[Benchmark]` method has no statements. The JIT will eliminate it.

**Fix:** Add code or return a value.

### NB0006 - Multiple baselines (Error)

**Cause:** Two or more methods in the same class have `[Benchmark(Baseline = true)]`.

**Fix:** Set `Baseline = true` on only one method per class.

### NB0007 - Duplicate lifecycle methods (Error)

**Cause:** Two methods in the same class share the same lifecycle attribute.

**Fix:** Remove the duplicate lifecycle method. Only one method per lifecycle attribute type is used (first discovered).

### NB0008 - [Benchmark] property out of range (Error)

**Cause:** `Iterations` or `WarmupIterations` on `[Benchmark]` is outside valid range.

Valid ranges: `Iterations` 0-100000, `WarmupIterations` 0-10000 (or sentinel -1 for unset).

### NB0009 - MeasurementOptions property out of range (Error)

**Cause:** `Iterations`, `WarmupIterations`, or `ConfidenceLevel` in a `MeasurementOptions` initializer or `with` expression is outside valid range.

Valid ranges: `Iterations` 0-100000, `WarmupIterations` 0-10000, `ConfidenceLevel` >0 and <1.

### NB0010 - Throwaway lambda body (Warning)

**Cause:** A lambda passed to `Benchmark.Run(Action)` or `Benchmark.RunRaw(Action)` has no observable side effects.

**Fix:** Use the `Func<T>` overload that returns a value, or add side effects.

```csharp
// Bad: JIT will eliminate this
Benchmark.Run(() => { var x = 42; });

// Good: return a value
Benchmark.Run(() => Compute());

// Also good: field write (side effect)
Benchmark.Run(() => { _result = Compute(); });
```

Only applies to `Action` (void) overloads. `Benchmark.Run<T>`, `RunAsync`, and `RunRaw` overloads accepting value-returning delegates are not flagged.

## Measurement Issues

### 0 ns results

**Causes:**

1. Dead code elimination (most common) - JIT removes the body
2. Dry-run mode active (`--dry-run`, or `Iterations=0` which skips measurement regardless of warmup count)

**Fixes:**

- Return a value from the benchmark body (use `Run<T>` / `RunAsync<T>`)
- Ensure the body has observable side effects
- Check for `--dry-run` flag or zero iterations in config

### Large Error / Wide Confidence Interval

**Causes:**

1. Too few iterations (default 200)
2. OS scheduling / context-switch noise
3. Thermal throttling on laptops

**Fixes:**

- Increase iterations: `.WithIterations(1000)` or `--iterations 1000`
- Switch outlier mode to `OutlierMode.IqrFence` for sporadic spikes
- Increase warmup: `.WithWarmup(50)` to let CPU stabilise
- Run plugged in (laptops throttle on battery)

### High StdDev

**Cause:** GC pressure or allocation noise.

**Fix:** Enable allocation tracking with `.WithAllocations()` to diagnose. If the benchmark intentionally exercises GC, disable `ForceGcBeforeEachIteration`.

### Significance shows "~" (not significant)

**Cause:** The Mann-Whitney U test ran but the p-value is >= 0.05, meaning no statistically significant difference was detected between this benchmark and the baseline.

Note: the significance threshold is hardcoded at p < 0.05. Changing `ConfidenceLevel` (e.g. to 0.99) affects the confidence interval on the mean, but does not change the significance threshold.

**Fix:** Increase iterations to reduce variance, or accept that the implementations are statistically indistinguishable at the current iteration count.

### Significance column is blank

**Cause:** The significance test could not run. This happens when there are fewer than 2 non-errored benchmarks in the run, when `EnableSignificance` is false, or when the Mann-Whitney U test has fewer than 5 samples per group (returns NaN p-value, verdict is `NotTested`).

**Fix:** Ensure at least 2 benchmarks are in the suite, `EnableSignificance` is true, and `MeasuredIterations` >= 5 per group.

### MarginOfError is ±0 ns

**Cause:** Only one sample (n < 2) or all measurements identical (timer resolution coarser than benchmark duration). The `StatsSummary` returns 0 for StdDev/StdErr/MarginOfError when n < 2.

**Fix:** Increase iterations. If the timer is too coarse, use a machine with higher resolution.

### Result shows zero allocations unexpectedly

**Cause:** `MeasureAllocations` is not enabled (default: false).

**Fix:** Set `WithAllocations(true)` or `new MeasurementOptions { MeasureAllocations = true }`.

## Discovery Issues

### Benchmark not discovered

**Causes:**

1. Method is `static` (only instance methods are discovered)
2. Class is `abstract`
3. Assembly not registered via `AddFromAssembly<T>()`
4. Wrong assembly scanned

**Fixes:**

- Use `--list` to verify what the host finds
- Ensure class is public, not abstract
- Ensure method is an instance method

### "Could not instantiate MyClass"

**Cause:** No public parameterless constructor. The host's `PerClassLifecycle.TryCreateInstance` uses `Activator.CreateInstance` by default, which requires a public parameterless constructor.

**Fixes:**

1. Add a public parameterless constructor
2. Use `WithInstanceFactory` for custom instantiation
3. Use `NBenchmark.DependencyInjection` package

### Benchmarks run in different order each time

**Cause:** Random order is the default (prevents systematic bias from CPU cache state, GC patterns).

**Fix:** Use `--order declaration` or `.WithRunOrder(RunOrder.Declaration)` for source order.

## Regression Detection

The `--threshold-pct <n>` flag fails the benchmark run if any benchmark's median exceeds baseline median by more than n percent.

```bash
dotnet run -- --threshold-pct 5
```

Median-based comparison: `candidate.Median > baseline.Median * (1 + pct/100)`. The baseline is the first explicit `IsBaseline` benchmark, or the one with the fastest median.

Sets `Environment.ExitCode = 1` on regression. Works in CI pipelines.

Zero-median baseline edge case: if baseline median is 0, any candidate with median > 0 is flagged as regressed.

## Configuration Tuning

### When to increase iterations

- High `CoefficientOfVariation` (>0.05)
- Wide confidence interval relative to mean
- Sub-microsecond operations need more iterations for stable results

### When to change outlier mode

- `RemoveTop5Percent` (default): good general-purpose
- `IqrFence`: sporadic large spikes from OS scheduling interrupts (adapts to data spread)
- `RemoveTopAndBottom5Percent`: very fast outliers (cache hits) also skew results
- `None`: latency-tail analysis where every sample matters

### When to increase warmup

- Thermal throttling on laptops (let CPU stabilise)
- JIT compilation is large (multiple generic instantiations)
- Memory-mapped I/O or lazy initialization happening during measurement

### GC strategy

- `ForceGcBeforeEachIteration = true` (default): clears GC heap before each warmup and measured iteration. Keep on unless the benchmark purposefully tests GC behaviour.
- `ForceGcBetweenBenchmarks = true` (default): full gen-2 collect between benchmarks in a suite. Keep on to prevent carry-over GC state.
- `MeasureAllocations = true`: enable to see allocations per operation. Has measurement overhead (thread-local GC stats).
