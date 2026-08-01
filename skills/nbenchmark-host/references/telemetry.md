# Telemetry Reference (OpenTelemetry / OTLP)

NBenchmark emits a full set of OpenTelemetry instruments through an internal `Meter` and `ActivitySource` (namespace `NBenchmark.Diagnostics`). The `--otlp-endpoint` CLI flag (or `OTEL_EXPORTER_OTLP_ENDPOINT` env var) wires an OpenTelemetry SDK in your entry assembly to export them. Use this reference when building Grafana dashboards, alerting rules, or custom trace viewers that consume NBenchmark telemetry.

> The `NBenchmarkDiagnostics` class itself is `internal`. The two stable seams it exposes are its `Meter` (name `"NBenchmark"`) and `ActivitySource` (name `"NBenchmark"`). A host running NBenchmark in-process can attach its own `MeterListener` / `ActivityListener` to these without depending on internal record methods. The standard path is `--otlp-endpoint`, which lets the OpenTelemetry SDK in your entry assembly export to a collector.

## Enabling telemetry

### CLI

```bash
dotnet run -c Release -- --otlp-endpoint http://localhost:4317
```

`--otlp-endpoint <url>` accepts an absolute `http://` or `https://` URL. The harness mirrors it into `OTEL_EXPORTER_OTLP_ENDPOINT` before launching workers, so isolated benchmarks stream to the same collector as the host.

### Environment variables

| Variable | Purpose |
|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Standard OTel exporter endpoint. Set by `--otlp-endpoint` when not already set. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Forwarded to children (`grpc` / `http/protobuf`). |
| `OTEL_EXPORTER_OTLP_HEADERS` | Forwarded to children (e.g. API keys). |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | Forwarded to children. |
| `OTEL_RESOURCE_ATTRIBUTES` | Standard OTel resource attributes (comma-separated `key=value`). Stamped on the root span. |
| `OTEL_SERVICE_NAME` | Standard OTel service name. Stamped on the root span as `service.name`. |
| `NBENCHMARK_OTEL_ENDPOINT` | The internal name for `--otlp-endpoint`. Forwarded to workers; mirrored into `OTEL_EXPORTER_OTLP_ENDPOINT` when the user hasn't set it. |

All six `OTEL_*` variables are forwarded to workers verbatim, so a worker streams to the same collector as the host. The in-memory `IMeasurementObserver` callback cannot cross the process boundary, so OTLP is the cross-process telemetry channel. For the observer stream that does cross (phase/detector/result events, and opt-in samples via `--stream-samples`), see [observers.md](../../nbenchmark-reporters/references/observers.md#isolated-worker-delivery).

## Metrics (Meter: `"NBenchmark"`)

### Histograms

| Instrument | Unit | Description | Tags |
|---|---|---|---|
| `nbenchmark.sample.duration` | `ns/op` | Per-op sample duration in nanoseconds | `benchmark`, `warmup` (bool), `phase` (string) |
| `nbenchmark.alloc.bytes_per_op` | `B/op` | Per-op allocation delta in bytes | `benchmark`, `warmup`, `phase` |

Recorded from the measurement thread via a value-type `TagList`; when no `MeterListener` is attached, `Record` short-circuits before touching the tags. The `phase` string is a cached literal from the caller (no per-sample allocation).

### Counters

| Instrument | Unit | Description | Tags |
|---|---|---|---|
| `nbenchmark.outliers.removed` | `samples` | Outlier samples removed | (none) |
| `nbenchmark.jitter.detector_switches` | `switches` | Outlier-detector auto-switches triggered by jitter | (none) |
| `nbenchmark.gc.gen0` | `collections` | Generation 0 GC collections during measurement | `benchmark` |
| `nbenchmark.gc.gen1` | `collections` | Generation 1 GC collections during measurement | `benchmark` |
| `nbenchmark.gc.gen2` | `collections` | Generation 2 GC collections during measurement | `benchmark` |

GC counters reflect the measurement-phase delta (computed by `DiagnosticMeter` as `after - before` across the measured loop) and are emitted post-run from `DiagnosticsResult`.

### Observable gauges

| Instrument | Unit | Description |
|---|---|---|
| `nbenchmark.ci.relative_half_width` | `ratio` | CI relative half-width of the running mean |
| `nbenchmark.jitter.metric` | `ratio` | Host jitter metric (MAD / median of calibration probes) |
| `nbenchmark.sample.mean_per_op` | `ns/op` | Running mean per-op duration from the measurement phase |
| `nbenchmark.ops_per_second` | `ops/s` | Running operations per second (`1e9 / mean per-op ns`) |
| `nbenchmark.samples.count` | `samples` | Running sample count |
| `nbenchmark.outliers.removed_total` | `samples` | Total outliers removed |

Gauges are process-wide and updated from the measurement thread. Use them for live dashboards (e.g. a CI dashboard showing the CI-width convergence curve in real time).

## Traces (ActivitySource: `"NBenchmark"`)

### Spans

| Span | Description | Tags |
|---|---|---|
| `benchmark.suite` | Root span for a `RunAsync` call | `nbenchmark.suite.name`, `nbenchmark.suite.benchmark_count`, `nbenchmark.profile`, `nbenchmark.runtime`, `nbenchmark.seed`, `nbenchmark.run_order`, plus all `TelemetryResource.Attributes` (see below) |
| `benchmark.run` | One benchmark method | `nbenchmark.name`, `nbenchmark.class`, `nbenchmark.baseline` (when true), `nbenchmark.parameter_set` (when parameterised) |
| `nbenchmark.phase.{phase}` | One phase (`jitter` / `calibration` / `warmup` / `measurement`) | `nbenchmark.benchmark.name`, `nbenchmark.phase` |

### Run-completed tags (on `benchmark.run`)

| Tag | Description |
|---|---|
| `nbenchmark.result.median_ns` | Final median (ns) |
| `nbenchmark.result.mean_ns` | Final mean (ns) |
| `nbenchmark.result.sample_count` | Post-trim sample count (`N`) |
| `nbenchmark.result.outliers_removed` | Outliers discarded |

### Phase-completed tags (on `nbenchmark.phase.{phase}`)

| Tag | Phase | Description |
|---|---|---|
| `nbenchmark.sample_stop_reason` | measurement | `CiTargetMet` / `MaxCeiling` / `ExplicitCount` / `WallClockCap` / `GraceCapExhausted` |
| `nbenchmark.warmup_stop_reason` | warmup | `Settled` / `MaxCeiling` / `ExplicitCount` / `WallClockCap` |
| `nbenchmark.resolved_k` | calibration | Resolved ops-per-sample |
| `nbenchmark.resolved_warmup` | warmup | Resolved warmup count |
| `nbenchmark.jitter_metric` | jitter | Host jitter metric |
| `nbenchmark.detector_switched` | jitter | `true` if the jitter run swapped the outlier detector (IqrFence -> MedianAbsoluteDeviation) |

### Span events

Discrete annotations on the phase span that explain why a phase ended. A trace UI renders these as markers on the flame-graph row.

| Event | Tags | When |
|---|---|---|
| `detector.switched` | `nbenchmark.from`, `nbenchmark.to`, `nbenchmark.jitter_metric` | Jitter run swapped the outlier detector |
| `warmup.plateau_reached` | (none) | Warmup ended via `Settled` |
| `measurement.ci_target_met` | `nbenchmark.achieved_ci_width`, `nbenchmark.ci_target` | Measurement stopped because the CI target was met |
| `phase.cap_hit` | (none) | Warmup or measurement hit a wall-clock or grace cap |

## Resource attributes (`TelemetryResource`)

`TelemetryResource.Attributes` (namespace `NBenchmark.Diagnostics`) is a static dictionary read once per process from environment variables and cached. It is stamped onto the root `benchmark.suite` span so a backend that joins on resource attributes sees them on every child span and metric.

### CI provider detection

| Attribute | Values | Env var trigger |
|---|---|---|
| `nbenchmark.ci_provider` | `github_actions` | `GITHUB_ACTIONS` |
| | `gitlab_ci` | `GITLAB_CI` |
| | `azure_pipelines` | `AZURE_PIPELINES` or `TF_BUILD` |
| | `circleci` | `CIRCLECI` |
| | `appveyor` | `APPVEYOR` |
| | `teamcity` | `TEAMCITY_VERSION` |
| | `jenkins` | `JENKINS_URL` |
| | `travis_ci` | `TRAVIS` |
| | `buildkite` | `BUILDKITE` |

### CI run identification

| Attribute | Env vars tried (first non-empty wins) |
|---|---|
| `nbenchmark.ci_run_id` | `GITHUB_RUN_ID`, `CI_PIPELINE_ID`, `BUILD_BUILDID`, `CIRCLE_BUILD_NUM`, `APPVEYOR_BUILD_ID`, `TEAMCITY_BUILDID`, `BUILDKITE_BUILD_ID`, `TRAVIS_BUILD_ID` |
| `nbenchmark.ci_run_url` | `GITHUB_SERVER_URL`, `CI_JOB_URL`, `BUILD_BUILDURI`, `CIRCLE_BUILD_URL` |
| `nbenchmark.ci_repository` | `GITHUB_REPOSITORY`, `CI_REPOSITORY_URL` |
| `nbenchmark.ci_ref` | `GITHUB_REF`, `CI_COMMIT_REF_NAME` |
| `nbenchmark.ci_attempt` | `GITHUB_RUN_ATTEMPT` |

### Git state

| Attribute | Env vars tried | Fallback |
|---|---|---|
| `nbenchmark.commit_sha` | `GITHUB_SHA`, `CI_COMMIT_SHA`, `GIT_COMMIT` | `git rev-parse --short HEAD` |
| `nbenchmark.branch` | `GITHUB_HEAD_REF`, `CI_COMMIT_BRANCH`, `GIT_BRANCH` | `git rev-parse --abbrev-ref HEAD` (skipped on detached HEAD) |

The git CLI fallback is best-effort: ignored on any failure (no repo, no git binary, timeout after 2s). A short SHA is used to keep the attribute compact.

### Host

| Attribute | Source |
|---|---|
| `nbenchmark.host.machine_name` | `Environment.MachineName` |
| `nbenchmark.host.os` | `windows` / `macos` / `linux` (from `OperatingSystem.*`) |
| `nbenchmark.host.arch` | `RuntimeInformation.ProcessArchitecture` (lowercased) |
| `nbenchmark.host.runtime` | `RuntimeInformation.FrameworkDescription` |

### OpenTelemetry standard

`OTEL_RESOURCE_ATTRIBUTES` (comma-separated `key=value` list) and `OTEL_SERVICE_NAME` are copied through verbatim, so a user who has already configured them for the rest of their service sees them on NBenchmark spans too. NBenchmark-specific attributes use the `nbenchmark.*` namespace to avoid collisions with the standard OTel schema.

## Reading telemetry in-process

For a host that runs NBenchmark in-process (e.g. `NBenchmark.Studio`), attach a `MeterListener` / `ActivityListener` to the public `Meter`/`ActivitySource` names:

```csharp
using System.Diagnostics.Metrics;

var listener = new MeterListener
{
    InstrumentPublished = (instrument, l) =>
    {
        if (instrument.Meter.Name == "NBenchmark")
            l.EnableMeasurementEvents(instrument);
    },
};
listener.SetMeasurementEventCallback<double>((inst, measurement, tags, state) =>
{
    // e.g. feed nbenchmark.sample.duration into a live histogram
});
listener.Start();
```

The `ActivitySource` name is `"NBenchmark"`; attach an `ActivityListener` the same way.

## Related references

- [cli.md](cli.md) - the `--otlp-endpoint` flag
- The `nbenchmark-reporters` skill and [observers.md](../nbenchmark-reporters/references/observers.md) - the in-process `IMeasurementObserver` seam (distinct from OTLP; does not cross the process boundary)
- The `nbenchmark-troubleshooting` skill - the jitter metric and detector-switch behaviour