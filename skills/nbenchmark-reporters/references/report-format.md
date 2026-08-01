# Report Format Versioning Reference

Every file reporter stamps two version numbers into its output, so a consumer storing NBenchmark results over time can tell whether two files may be compared. NBenchmark does not read its own reports back - nothing here is used to gate an internal comparison. The stamps exist entirely for whoever stores the files: a CI trend dashboard, a regression script, a spreadsheet.

The constants live on `NBenchmark.Reporters.ReportFormat`:

```csharp
public static class ReportFormat
{
    public const int SchemaVersion = 1;
    public const int MeasurementEpoch = 3;
}
```

## Two numbers, because two independent things change

| Stamp | Answers | Bump when | Do NOT bump for |
|---|---|---|---|
| `SchemaVersion` | Can a consumer still **parse** this file? | Renaming or removing a field, changing a field's type, restructuring the envelope | Adding an optional field (a consumer that ignores unknown fields is unaffected) |
| `MeasurementEpoch` | Can a consumer still **compare** its numbers? | Harness overhead changes (dispatch, allocation measurement, the timing loop); the default runtime profile or its knobs change; the definition of a reported statistic changes (what counts as an outlier, how ns/op is derived) | New fields, reporter formatting, or fixes that leave the numbers where they were |

Conflating the two means either silently breaking parsers or silently plotting a step change as a regression. An absent stamp is not epoch 0 - it means the file predates the concept, and nothing is known about its comparability. Consumers should reject such files rather than assume them equivalent to the earliest declared epoch.

## Where each reporter stamps them

### JSON

The envelope opens with both fields:

```json
{
  "generatedAt": "2026-07-31T20:51:25+00:00",
  "schemaVersion": 1,
  "measurementEpoch": 3,
  "detail": "simple",
  "results": [ ... ]
}
```

JSON always carries the full record regardless of `--detail`. The schema/epoch fields are the first two after the timestamp.

### Markdown

A header line above the table:

```
> Runtime: **steady-state** (tiered=off pgo=off r2r=off)
> Format: schema 1, measurement epoch 3 (numbers are comparable only with the same epoch)
```

The `Runtime:` line records the runtime profile (see the [isolation reference](../nbenchmark/references/isolation.md#runtime-profiles)).

### CSV

`SchemaVersion` and `MeasurementEpoch` columns appear at every detail level (Simple, Standard, Advanced), alongside `RuntimeProfile` and `RuntimeKnobs`:

| Detail | Additional ratio columns |
|---|---|
| Simple | (none - base ratio only) |
| Standard / Advanced | `RatioCiLower`, `RatioCiUpper`, `RatioReplicates` (the paired ratio interval; see [significance-and-outliers.md](../nbenchmark/references/significance-and-outliers.md#paired-ratio-estimation)) |

## Measurement epoch history

Each epoch is a one-line marker for "a stored baseline covering [these cases] is not comparable across this point":

| Epoch | What moved |
|---|---|
| 1 | First declared epoch. Monomorphic dispatch (no per-op boxing for value-returning benchmarks), suites isolated in worker processes by default under the `steady-state` runtime profile. The typed-delegate refactor moved the calibration standard from 9.34 ns / 24 B per op to 2.53 ns / 0 B - the motivating case for this counter. |
| 2 | The multi-launch reporting overhaul. A multi-launch benchmark reports the **average** of its launches (not the fastest); its interval comes from the spread **between** launches (not within one); the ratio is the geometric mean of the per-launch ratios (not the quotient of two aggregated medians). `--threshold-pct` gates on the paired value, so a gate can change verdict on unchanged code. |
| 3 | Shapes that previously fell back to the host are now measured in a worker: parameter sweeps, suite and per-iteration lifecycle, custom statistical strategies built with constructor arguments, and DI-resolved benchmark instances. A row that was already isolated reports the same number; a row that was not moves by however much the host's JIT tiering was worth (up to 3.3x on bodies of provably identical cost). |

## Consuming the stamps

A consumer storing results for trend analysis or regression gating should:

1. Read both stamps before comparing any two files.
2. Reject files with no stamp ("unknown comparability") rather than treating them as epoch 0.
3. Refuse to compare files with different `MeasurementEpoch` values - the numbers describe a different measurement regime.
4. Refuse to parse files with a `SchemaVersion` newer than the consumer understands.
5. Record the stamps alongside stored results so a later audit can reconstruct which regime produced which number.

### Python consumer example

```python
import json
from pathlib import Path

CURRENT_SCHEMA = 1
CURRENT_EPOCH = 3

def load_results(path: Path):
    with path.open() as f:
        env = json.load(f)
    schema = env.get("schemaVersion")
    epoch = env.get("measurementEpoch")
    if schema is None or epoch is None:
        raise ValueError(f"{path}: pre-versioning file; comparability unknown")
    if schema > CURRENT_SCHEMA:
        raise ValueError(f"{path}: schema {schema} newer than reader knows")
    if epoch != CURRENT_EPOCH:
        raise ValueError(
            f"{path}: epoch {epoch} differs from current {CURRENT_EPOCH}; "
            "numbers are not comparable")
    return env["results"]
```

## Related references

- [isolation.md](../nbenchmark/references/isolation.md) - runtime profiles and `RuntimeProfileName`/`RuntimeKnobs`
- [significance-and-outliers.md](../nbenchmark/references/significance-and-outliers.md) - the paired ratio interval columns
- The `nbenchmark-reporters` skill - how to attach reporters and choose detail levels