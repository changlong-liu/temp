# Foray Generated Script Optimization Report

## Summary

This report covers direct optimization of the following generated SCOPE scripts:

- `C:\temp\temp\aa\12c543ef-123d-44f5-9ee6-f63b9fe05f27.script` (Search QoS script)
- `C:\temp\temp\aa\1ab1998c-bbff-4c2e-b54e-f8994942d2bc.script` (BizChat script)

The work preserved all output metrics, segments, output paths, Foray metadata, union ordering, histogram outputs, and
stream expiry settings. Because output compatibility was required, no final metric or segment was removed. The first
requested optimization was therefore applied only to dead intermediate expressions and columns.

The main measurable improvement is in the BizChat script: `JObject.Parse` calls were reduced from 175 to 107, including
a reduction in direct `R9Latency` parsing from 59 calls to 13. Both original scripts were saved in the Copilot session
workspace before modification.

## Scope and constraints

The scripts were optimized directly rather than by changing their Foray MetricSet or upstream Views. This has two
important consequences:

1. A future Foray or Mangrove generation can overwrite these changes.
2. Filtering can be made explicit before expensive expressions in the script, but physical pushdown still depends on
   the SCOPE optimizer and View implementation.

The output contract was treated as immutable. In particular:

- All 301 metrics in the Search QoS script were retained.
- All 222 unique unpivoted metrics in the BizChat script were retained.
- All existing segments and `AggregateAll` behavior were retained.
- Native, Bootstrap, and Histogram output schemas were retained.

## Baseline

| Measurement | Search QoS script | BizChat script |
|---|---:|---:|
| Original size | 305,219 bytes | 2,471,789 bytes |
| Unique metrics | 301 | 222 |
| `JObject.Parse` calls | 0 | 175 |
| `JArray.Parse` calls | 0 | 160 |
| Bootstrap references | 0 | 390 |
| Histogram processors | 0 | 2 |

Original SHA-256 hashes:

| Script | SHA-256 |
|---|---|
| Search QoS script | `4A66B9F1073F887F9A524C1D6C79A3B8C89A61FA3F8261B3B1341D60C6999FC1` |
| BizChat script | `6F9D392AAC7C7F0BFAEEDCBF6720C3A135310968CB7D47169CF7CB728DC2C554` |

## Implemented optimizations

### 1. Remove dead intermediate work

#### Search QoS script

Five values in the first Search QoS projection were not consumed by `Base` or any later stage:

- A duplicate raw `Scopes` projection.
- Two generated `filterCondition...` aliases that repeated predicates already applied by `WHERE`.
- A standardized `FLIGHT` value that was recalculated from the retained source Flight value in `Base`.
- A formatted `REQUEST_TIME` value that was recalculated from the retained timestamp in `Base`.

These expressions were removed. All metric-vectorization inputs, reducer keys, rollup dimensions, and output columns
remain unchanged.

#### BizChat script

Nine generated ternary expressions whose true and false branches both returned `1` were folded to the constant `1`.
This removes unnecessary null, empty-string, and state checks without changing their values.

No final metric or segment was removed because the requested compatibility policy requires the complete output schema.

### 2. Reduce repeated JSON and collection parsing

Repeated JSON parsing in the BizChat script was consolidated within individual expressions by parsing once and reusing
the
parsed token through a single-element LINQ projection. The optimized areas include:

- `R9Latency` measure extraction.
- `R9Latency` tag extraction.
- Performance-segment classification.
- `ProductInformation` user and license classification.
- `RealtimeTurnResult` checks.
- Dataverse latency extraction.
- Mobile chat latency fallback extraction.
- Bootstrap statistic expressions that repeatedly parsed the same JSON value.

The changes preserve existing behavior for empty values, missing properties, malformed JSON, and invalid numeric
values. Malformed JSON and invalid conversions continue to throw; no broad exception handler or silent fallback was
introduced.

Three expressions that previously used `Any()` followed by a repeated parse and `First()` now parse once and select
the first matching nullable value. Their no-match result remains `null`, and their first-match ordering is unchanged.

### 3. Limit Bootstrap and Histogram work

The Turn- and Session-level Bootstrap and Histogram graphs were traced from both final outputs. Every existing
Bootstrap statistic and histogram column has a downstream consumer in either the main output or
`__histogram.txt`.

Removing any of this work would change the required output contract. Consequently, no Bootstrap or Histogram metric
was deleted. Common expressions were simplified where they already occurred inside a single statistic expression,
but cross-column sharing was not introduced without a SCOPE compiler and the referenced runtime assemblies.

This item produced an audit result rather than a structural branch removal.

### 4. Apply early filtering and narrow projections

The BizChat script now has an explicit `FilteredInput` rowset before generated metric and JSON expressions. It:

- Applies the existing application, scenario, Flight, and date predicates before the large metric projection.
- Projects 58 source columns required by `ROWSET1`.
- Excludes unused source columns from the generated metric pipeline.

A lexical column-closure check found 58 consumed source columns and 58 projected columns, with no missing or extra
columns. `Scenario` was explicitly included because it is required by both the source predicate and downstream
metrics.

The Search QoS script already applied its date, Flight, and randomization-ID predicates directly against the source
View. Adding
another rowset boundary could introduce an extra stage, so its existing filter placement was retained. Its first
projection was narrowed by removing the five dead values described above.

## Results

| Measurement | Search before | Search after | BizChat before | BizChat after |
|---|---:|---:|---:|---:|
| File size | 305,219 | 304,261 | 2,471,789 | 2,469,604 |
| Bytes removed | - | 958 | - | 2,185 |
| `JObject.Parse` | 0 | 0 | 175 | 107 |
| `JArray.Parse` | 0 | 0 | 160 | 156 |
| Direct `R9Latency` parses | 0 | 0 | 59 | 13 |

The `JObject.Parse` count in the BizChat script decreased by approximately 38.9%. The remaining calls occur across
independent
output expressions or collection elements where safe cross-column reuse would require a typed preprocessing processor
and compiler validation.

Optimized SHA-256 hashes:

| Script | SHA-256 |
|---|---|
| Search QoS script | `28616ED295542F00BFAB137B64AD7E91C058E50C9E98746F35964B7D45F16029` |
| BizChat script | `F9C0A9C17D9F8F8D887A26DCACDBDCC4B1558248EF4B2EBE80FD5E13C324BB2D` |

## Compatibility checks

Static comparisons confirmed that the following contracts are unchanged:

- Output paths and output count.
- Main and histogram unpivot metric lists and ordering.
- Segment definitions and ordering.
- Profile configuration.
- Foray metadata headers.
- Native and Bootstrap union composition.
- Histogram processor count.
- Histogram stream expiry.
- Search QoS metric-dimension mapping and reducer keys.

The optimized BizChat script has no IDE diagnostics. The scripts could not be compiled or executed locally because
`C:\temp\temp\aa` does not contain their `References` and `Resources` directories. Access to the production Search QoS
View was also unavailable from the local environment.

No runtime-performance claim is made. The static reductions should reduce repeated JSON parsing and intermediate row
width, but job-level CPU, memory, vertex count, shuffle volume, and latency must be measured with matched executions.

## Remaining opportunities

For durable and larger improvements, the same transformations should be implemented in the Foray MetricSet generator
or upstream Views:

1. Parse `R9Latency`, `ProductInformation`, action arrays, and tool arrays into typed columns once per input row.
2. Generate only columns transitively required by selected profiles and metrics.
3. Emit shared preprocessing stages for Native and Bootstrap paths.
4. Make Bootstrap selection explicit in the MetricSet instead of expanding then pruning generated code.
5. Benchmark the fixed 4,000-partition Search QoS configuration using actual input volume and skew telemetry.

These changes were not made here because the selected implementation scope was limited to the two generated scripts.
