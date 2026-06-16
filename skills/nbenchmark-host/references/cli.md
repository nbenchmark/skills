# CLI Reference (Host mode)

`BenchmarkHost.Create(args)` parses these flags automatically. Usage:

```bash
dotnet run -- [options]
# or a published binary:
MyApp.Benchmarks [options]
```

## Flags

### `--filter <pattern>`

Run only benchmarks whose fully-qualified name (`ClassName.MethodName`) matches the glob. `*` matches any sequence; matching is case-insensitive. A class with no matching methods is skipped.

```bash
dotnet run -- --filter String*
dotnet run -- --filter *.Contains*
dotnet run -- --filter StringBenchmarks.Concat
```

### `--iterations <n>`

Pin the measured-sample count per benchmark, disabling auto-sampling. Range `0`–`100,000`. Default **auto** (sampling stops when the CI meets `--ci-target`). Prefer `--dry-run` over `--iterations 0`.

### `--warmup <n>`

Pin the warmup-sample count per benchmark, disabling plateau detection. Range `0`–`10,000`. Default **auto**.

### Adaptive tuning flags

When `--iterations` / `--warmup` / `--ops-per-sample` are left unset, the adaptive loop resolves them. These flags bound and steer it.

| Flag | Default | Effect |
|---|---|---|
| `--auto-tune <preset>` | `default` | Preset bundle: `default`, `quick` (fewer samples, ±5% CI), or `thorough` (more samples, ±1% CI). |
| `--ops-per-sample <n>` | auto | Pin K — body invocations timed as one sample. Range `1`–`16,777,216`. Auto-calibrated otherwise. |
| `--ci-target <0-1>` | `0.025` | Target relative CI half-width for auto-sampling; sampling stops once met. |
| `--min-samples <n>` | `30` | Floor on auto-resolved measured samples. |
| `--max-samples <n>` | `100000` | Ceiling on auto-resolved measured samples. |
| `--min-warmup <n>` | `8` | Floor on auto-detected warmup samples. |
| `--max-warmup <n>` | `10000` | Ceiling on auto-detected warmup samples. |
| `--max-tuning-time <s>` | `20` | Per-benchmark wall-clock safety cap (seconds) for the whole loop. |

```bash
dotnet run -- --auto-tune quick
dotnet run -- --auto-tune thorough --max-tuning-time 60
dotnet run -- --ops-per-sample 256 --ci-target 0.01
```

### `--confidence <value>`

Confidence level for the Error column. Decimal strictly between `0` and `1`. Default `0.95`.

### `--alpha <value>`

Significance level (alpha) for the Mann-Whitney U test. A benchmark is flagged significant when its p-value is below this. Decimal strictly between `0` and `1`. Default `0.05`.

```bash
dotnet run -- --alpha 0.01
```

### `--reporter <type>`

Add a reporter by name; repeatable to stack reporters.

| Name | Reporter | Output |
|---|---|---|
| `json` | `JsonReporter` | JSON file in `--output` dir (or CWD) |
| `markdown` | `MarkdownReporter` | Markdown file in `--output` dir (or CWD) |
| `csv` | `CsvReporter` | CSV file in `--output` dir (or CWD) |
| `console` | `ConsoleReporter` | Terminal table (requires `NBenchmark.Reporters.Console`) |

The `console` reporter self-registers when the package is referenced. Reporters from external packages register the same way: reference the package, then use `--reporter <name>`.

```bash
dotnet run -- --reporter markdown
dotnet run -- --reporter json --reporter csv
```

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
| `advanced` | Adds a per-benchmark stats block (quartiles, fences, CI, skewness, kurtosis, MAD, allocation percentiles) |

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

### `--threshold-pct <n>`

Exit with **code 1** if any benchmark regresses more than `n`% against the baseline (`n` ≥ 1). A benchmark regresses when `candidate.Median / baseline.Median > 1.0 + n/100`. If the baseline median is `0`, any non-baseline benchmark with a positive median is treated as regressed. Baseline = the `[Benchmark(Baseline = true)]` method, or the fastest median if none. Errored benchmarks are excluded.

```bash
dotnet run -- --threshold-pct 10
```

### `--help` / `-h`

Print help text and exit.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Run completed. Errored benchmarks are recorded but not fatal. |
| `1` | An argument error (unknown flag, missing value, out-of-range `--iterations`/`--warmup`/`--ops-per-sample`/`--ci-target`/`--min-samples`/`--max-samples`/`--min-warmup`/`--max-warmup`/`--max-tuning-time`, bad `--confidence`/`--alpha`/`--seed` format, unknown `--auto-tune` preset, unknown `--reporter`, invalid `--detail`), **or** a `--threshold-pct` regression. |

Even when exit code `1` is set, the run still completes — discovery, measurement, and reporting proceed — so you see output. On a `--threshold-pct` regression, reporters still flush their output. The non-zero code ensures CI catches the problem.

## CI examples

```bash
# Fail the build if anything is >10% slower than baseline; keep a markdown report
dotnet run -c Release -- --threshold-pct 10 --reporter markdown --output ./results

# Reproducible run in declaration order
dotnet run -- --order declaration --seed 12345

# Validate discovery and DI wiring without measuring
dotnet run -- --list
dotnet run -- --dry-run
```

```yaml
# GitHub Actions step
- name: Benchmarks
  run: >
    dotnet run -c Release --project benchmarks --
    --threshold-pct 10 --reporter json --output ./bench-results
```
