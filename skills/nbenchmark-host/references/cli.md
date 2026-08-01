# CLI Reference (Harness mode)

`BenchmarkHarness.Create(args)` parses these flags automatically. Usage:

```bash
dotnet run -- [options]
# or a published binary:
MyApp.Benchmarks [options]
```

If you use the standalone `NBenchmark.Tool` global tool (`dotnet benchmark`), it also accepts `--project <path>` (builds a .csproj with `dotnet build -c Release` and benchmarks the resulting DLL — picks the best TFM match) and `--assembly <path>` (benchmarks a pre-built DLL). With neither, it auto-discovers `*.dll` in the current directory. All other flags are forwarded to `BenchmarkHarness.Create`.

## Flags

### `--filter <pattern>`

Run only benchmarks whose fully-qualified name (`ClassName.MethodName`) matches the glob. `*` matches any sequence; matching is case-insensitive. A class with no matching methods is skipped.

```bash
dotnet run -- --filter String*
dotnet run -- --filter *.Contains*
dotnet run -- --filter StringBenchmarks.Concat
```

### `--category <name>` / `--exclude-category <name>`

Include or exclude benchmarks tagged with `[BenchmarkCategory]`. Repeatable; multiple `--category` flags are OR'd together, multiple `--exclude-category` flags are OR'd together.

### `--iterations <n>`

Pin the measured-sample count per benchmark, disabling auto-sampling. Range `0`-`100,000`. Default **auto** (sampling stops when the CI meets `--ci-target`). Prefer `--dry-run` over `--iterations 0`.

### `--warmup <n>`

Pin the warmup-sample count per benchmark, disabling plateau detection. Range `0`-`10,000`. Default **auto**.

### Adaptive tuning flags

When `--iterations` / `--warmup` / `--ops-per-sample` are left unset, the adaptive loop resolves them. These flags bound and steer it.

| Flag | Default | Effect |
|---|---|---|
| `--auto-tune <preset>` | `default` | Preset bundle: `default`, `quick` (fewer samples, looser CI), or `thorough` (more samples, tighter CI). |
| `--ops-per-sample <n>` | auto | Pin K — body invocations timed as one sample. Range `1`-`16,777,216`. Auto-calibrated otherwise. |
| `--ci-target <0-1>` | `0.025` | Target relative CI half-width for auto-sampling; sampling stops once met. |
| `--min-samples <n>` | `30` | Floor on auto-resolved measured samples. |
| `--max-samples <n>` | `100000` | Ceiling on auto-resolved measured samples. |
| `--min-warmup <n>` | `8` | Floor on auto-detected warmup samples. |
| `--max-warmup <n>` | `10000` | Ceiling on auto-detected warmup samples. |
| `--max-tuning-time <s>` | `20` | Per-benchmark wall-clock safety cap (seconds) for the whole loop. |
| `--autotune-cap-behavior <mode>` | `warn` | Cap handling: `warn` or `error`. |
| `--warmup-budget-fraction <0-1>` | `0.4` | Max share of `--max-tuning-time` for calibration + warmup. |
| `--cap-grace-factor <n>` | `1.5` | Multiplier on `--max-tuning-time` the measurement phase may reach while chasing `--min-samples`. |
| `--min-warmup-time <ms>` | `100` | Minimum wall-clock warmup time before auto-warmup may settle (0 disables). |
| `--no-jit-quiescence` | off | Disable the JIT-quiescence warmup gate (keep only the time floor). |

```bash
dotnet run -- --auto-tune quick
dotnet run -- --auto-tune thorough --max-tuning-time 60
dotnet run -- --ops-per-sample 256 --ci-target 0.01
```

### `--launch-count <n>`

Repeat each benchmark as `n` separate launches (worker processes), populating `BenchmarkResult.LaunchStatistics` (per-launch medians, mean, stddev, CI, and the reproducibility diagnostics `ProcessVarianceRatio` / `BetweenLaunchDispersion`). Range `1`-`100` (see the `LaunchCounts` static class). Harness mode defaults to **3** (Single/Suite modes default to 1), so the cross-launch interval surfaces without users asking. With two or more launches, the ratio gate evaluates the **paired** per-replicate ratio (geometric mean with a multiplicative CI) instead of a ratio of aggregated medians. Per-method `[Benchmark(LaunchCount = n)]` overrides; isolated groups take the max across members.

`--launch-count` has no effect under `--in-process` - in-process runs cannot have multiple worker launches, so the count is pinned to 1.

### `--confidence <value>`

Confidence level for the Error column. Decimal strictly between `0` and `1`. Default `0.95`.

### `--alpha <value>`

Significance level (alpha) for the significance test. A benchmark is flagged significant when its p-value is below this. Decimal strictly between `0` and `1`. Default `0.05`.

```bash
dotnet run -- --alpha 0.01
```

### `--min-practical-effect <0-1>`

Min practical effect (Cliff's δ) for a significant verdict. Default `0.147` (a "small" effect). `0` = p-value only; omit or set to `null` via code to disable the gate entirely.

### `--outlier <mode>`

Outlier trimming strategy. Values: `none`, `top5` (RemoveTop5Percent), `both5` (RemoveTopAndBottom5Percent), `iqr` (IqrFence, **default**), `mad` (MedianAbsoluteDeviation).

### `--tail-basis <basis>`

Source for percentile / min / max / histogram: `raw` (the full pre-trim distribution, **default**) or `trimmed` (kept samples only).

### `--percentiles <list>`

Custom percentile values to compute and report, comma-separated. E.g. `0.50,0.95,0.99,0.999`. Default `0.50,0.95,0.99,0.999,1.0`.

### `--no-histogram`

Disable latency histogram computation.

### `--reporter <type>`

Add a reporter by name; repeatable to stack reporters.

| Name | Reporter | Output |
|---|---|---|
| `json` | `JsonReporter` | JSON file in `--output` dir (or CWD) |
| `markdown` | `MarkdownReporter` | Markdown file in `--output` dir (or CWD) |
| `csv` | `CsvReporter` | CSV file in `--output` dir (or CWD) |
| `console` | `ConsoleReporter` | Terminal table (requires `NBenchmark.Reporters.Console`) |

The `console` reporter self-registers when the package is referenced. Reporters from external packages register the same way: reference the package, then use `--reporter <name>`. Auto-attached reporters (registered via `ReporterRegistry.RegisterAutoAttach`) are added automatically even when `--reporter` is not specified.

```bash
dotnet run -- --reporter markdown
dotnet run -- --reporter json --reporter csv
```

### `--observer <type>`

Attach a measurement observer by name (repeatable; multiple observers are composed into a fan-out). Resolved through `ObserverRegistry`. Auto-attached observers (registered via `ObserverRegistry.RegisterAutoAttach`) are added automatically.

### `--output <directory>`

Output directory for file reporters. **Must be under the current working directory.** Created automatically. Default: current directory.

```bash
dotnet run -- --reporter markdown --output ./results
```

### `--order <mode>`

| Value | Behaviour |
|---|---|
| `random` | Fisher-Yates shuffle, random seed each run **(default)** |
| `declaration` | Run in method-declaration order |

### `--seed <n>`

Fixed integer seed for reproducible random ordering. No effect with `--order declaration`.

### `--detail <level>`

| Value | Behaviour |
|---|---|
| `simple` | 10-column table with essential statistics **(default)** |
| `standard` | Adds percentile columns, effect size, CI bounds, margin %, outliers removed |
| `advanced` | Adds a per-benchmark stats block (quartiles, fences, CI, skewness, kurtosis, MAD, allocation percentiles, auto-tune) |

Affects all registered reporters. JSON always emits the full record regardless. See the `nbenchmark-reporters` skill for the column reference.

### `--list`

List discovered benchmarks without running them.

```
── StringBenchmarks ──
    Concat - current production implementation
    Interpolate
── DatabaseBenchmarks ──
    RunQuery
```

### `--dry-run`

Skip measurement entirely. Equivalent to `--iterations 0 --warmup 0`: classes are discovered, setup/teardown wired up, instances created — but the body is never invoked. For a one-shot smoke test, use `--iterations 1 --warmup 0` instead.

### `--in-process`

Run every benchmark in the host process - the opt-out from the isolated-by-default execution model. Each benchmark normally runs in a dedicated worker process; `--in-process` (or `[InProcess]` on a method, or `WithIsolation(false)` in code) disables that for the run. `--dry-run` is always in-process. See the [isolation reference](../../nbenchmark/references/isolation.md) for the worker model and the capture fallback.

### `--strict-isolation`

Fail the run (exit code 1) if any benchmark was not isolated. Failures are grouped by cause with remedies - the capture fallback, a missing worker, an unaddressable plan. The advisory `Iso` column nobody reads is indistinguishable from none; this makes isolation enforceable in CI.

```bash
dotnet run -c Release -- --strict-isolation --reporter markdown --output ./results
```

### `--verify-isolation`

Re-measure every benchmark in-process and print how much isolation changed it, as diagnostics only. Publishes nothing. Skipped under `--runtimes` (this process is one runtime, so the comparison would describe a different runtime's worker rather than the host). The per-benchmark delta shows the cost - or benefit - of the host's inherited runtime configuration versus a clean worker.

### `--runtime-profile <name>`

The runtime-startup configuration to measure under: `steady-state` (**default**), `production`, `server-gc`, or `host`. Applied to workers through their environment block at launch; in-process benchmarks inherit the host's configuration and report `host`. See [isolation.md](../../nbenchmark/references/isolation.md#runtime-profiles) for what each profile sets and when to use it.

### `--stream-samples`

Forward the live per-sample observer stream (`IMeasurementObserver.OnSample`) across the worker boundary, in batches of 128 samples or 100 ms - whichever comes first. Phase transitions, detector snapshots, and results cross unconditionally; this is the opt-in for the one channel whose cost scales with how fast the benchmarked code is. Withdrawn automatically when no observer is attached. Second and later replicates of a multi-launch run do not forward telemetry. In-process runs are unaffected (the observer is called directly). See [observers.md](../../nbenchmark-reporters/references/observers.md#isolated-worker-delivery).

### `--emit-raw`

Lift the 4096-sample cap on raw samples returned from an isolated worker - return every sample. The programmatic equivalents are `MeasurementOptions.MaxRawSamples` and `MeasurementOptions.UnboundedRawSamples`. This bounds only what crosses the process boundary; every reported statistic is computed inside the worker over the complete array, so raising it cannot move a median, interval, or outlier count. See [measurement-options.md](../../nbenchmark/references/measurement-options.md#maxrawsamples).

### `--cross-class`

Compute significance across all classes instead of per-class (the default). Equivalent to `WithCrossClassSignificance(true)`.

### `--runtimes <list>`

Runtimes to compare, comma-separated (e.g. `net8,net9,net10` or `net8.0,net9.0,net10.0`). Each runtime builds and runs in its own worker process. `--runtimes` overrides `--in-process`. Valid values: `net8`, `net9`, `net10`. Suite-mode multi-runtime runs require a `[BenchmarkPlan]` factory rather than an inline suite, because benchmark bodies are addressed by metadata token which is only valid within the build that produced it. See [isolation.md](../../nbenchmark/references/isolation.md#benchmarkplan---the-escape-hatch).

### `--profile <mode>`

Measurement profile: `realistic` (**default**) or `independent`. `Realistic` matches production (no forced GC between iterations, warmup heap carries into measurement). `Independent` forces a Gen0 GC before each iteration and a full GC between warmup and measurement.

### `--force-gc`

Force a Gen0 GC before every iteration (overrides profile). Equivalent to `ForceGcBeforeEachIterationOverride = true`.

### `--no-allocations`

Disable allocation tracking (overrides profile). Equivalent to `MeasureAllocationsOverride = false`.

### `--no-gc-between-benchmarks`

Disable the full GC between benchmarks (on by default for both profiles).

### `--diagnostics <mode>`

Runtime diagnostics: `none`, `gc` (default, GC collection counts), `gcandcpu` (GC + CPU time), `all` (GC + heap + exceptions + CPU time).

### `--cpu-affinity <list>`

Pin the benchmark process to logical CPU cores, comma-separated (e.g. `0` or `2,3`). Not supported on macOS (warning, ignored).

### `--priority <level>`

Process priority: `normal`, `idle`, `belownormal`, `abovenormal`, `high`, `realtime`.

### `--dedicated-host-guidance`

Warn when the host looks noisy (low core count, unraisable priority, macOS throttling). Off by default.

### `--otlp-endpoint <url>`

OTLP endpoint for the OpenTelemetry SDK (`http://` or `https://`); forwarded to workers so they stream to the same collector as the host.

### `--threshold-pct <n>`

Exit with **code 1** if any benchmark regresses more than `n`% against the baseline (`n` >= 1). A benchmark regresses when the ratio of candidate to baseline exceeds `1.0 + n/100`. If the baseline median is `0`, any non-baseline benchmark with a positive median is treated as regressed. Baseline = the `[Benchmark(Baseline = true)]` method, or the fastest median if none. Errored benchmarks are excluded.

When launch data is available (the Harness default is 3 launches), the gate evaluates the **paired** per-replicate ratio - the geometric mean of per-launch ratios with a multiplicative confidence interval - instead of a ratio of aggregated medians. The two can disagree, which is exactly the case the pairing exists for. See [significance-and-outliers.md](../../nbenchmark/references/significance-and-outliers.md#what-gates-on-it).

```bash
dotnet run -- --threshold-pct 10
```

### `--help` / `-h`

Print help text and exit.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Run completed. Errored benchmarks are recorded but not fatal. |
| `1` | An argument error (unknown flag, missing value, out-of-range `--iterations`/`--warmup`/`--ops-per-sample`/`--ci-target`/`--min-samples`/`--max-samples`/`--min-warmup`/`--max-warmup`/`--max-tuning-time`/`--launch-count`, bad `--confidence`/`--alpha`/`--min-practical-effect`/`--seed` format, unknown `--auto-tune`/`--autotune-cap-behavior` preset, unknown `--reporter`/`--observer`, invalid `--detail`/`--profile`/`--outlier`/`--tail-basis`/`--diagnostics`/`--priority`/`--runtime-profile`, bad `--cpu-affinity`/`--percentiles`/`--runtimes`/`--otlp-endpoint` value), a `--threshold-pct` regression, **or** a `--strict-isolation` failure (one or more benchmarks could not be isolated). An errored benchmark is not counted as a strict-isolation failure - it failed to measure, not failed to isolate. |

Even when exit code `1` is set, the run still completes — discovery, measurement, and reporting proceed — so you see output. On a `--threshold-pct` regression, reporters still flush their output. The non-zero code ensures CI catches the problem.

## CI examples

```bash
# Fail the build if anything is >10% slower than baseline; keep a markdown report
dotnet run -c Release -- --threshold-pct 10 --reporter markdown --output ./results

# Fail the build if any benchmark could not be isolated (CI enforcement of the default)
dotnet run -c Release -- --strict-isolation --reporter json --output ./results

# Reproducible run in declaration order
dotnet run -- --order declaration --seed 12345

# Validate discovery and DI wiring without measuring
dotnet run -- --list
dotnet run -- --dry-run

# Cross-runtime comparison with full diagnostics
dotnet run -c Release -- --runtimes net8,net10 --diagnostics all --detail advanced

# Pin to cores 2-3, raise priority, warn on shared host
dotnet run -- --cpu-affinity 2,3 --priority high --dedicated-host-guidance
```

```yaml
# GitHub Actions step
- name: Benchmarks
  run: >
    dotnet run -c Release --project benchmarks --
    --threshold-pct 10 --reporter json --output ./bench-results
```