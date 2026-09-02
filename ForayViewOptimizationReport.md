# Foray View Optimization Report

## Scope and result

The downloaded dependency chains for both jobs were analyzed:

- BizChat: Fusion metric View, resolved Copilot Data Fusion View, usage/score join path, flight View, and resolver
  wrappers.
- Search QoS: Metric View, raw-log View, Logical Impression Views, flight View, and resolver wrappers.

Two output-compatible optimizations were applied. No public View schema, parameter, filter result, join result, script
output, partition hint, row-count hint, flight rule, or feature flag was changed.

## Applied optimizations

### BizChat resolved score path

`CopilotDataFusionView_resolved_0.view` now:

- Filters empty `RequestId` score records before wide score normalization and five `SSRMetrics` JSON extractions.
- Removes the unused `ScoreDataNotToJoin` rowset.
- Emits constant `false` for `IsOnlyScoreData`.

The constant is equivalent because this path is a left join from UsageData, whose `NormalizedRequestId` is always a
normalized string or `string.Empty`, never null. Empty score request IDs could not join and were already removed later.

This file is a resolved job artifact. The durable implementation should be made in the source Copilot Data Fusion
module that generates it.

### Search abandoned-session predicate

`MetricSet_MSAI_SubstrateSearch_QoS_Usage.view` now uses `Any(...)` instead of
`Where(...).Count() > 0` when checking for non-`sessionend` actions. This short-circuits after the first match and avoids
allocating a lowercase event-type string.

The target Search job disables this branch with `RemoveSingleSuggestionSessions=false`; the optimization benefits
callers that enable it without changing the target job's active path.

## Active-path observations

### BizChat

- The 4,000-vertex hot merge stage corresponds to the wide UsageData-to-ScoreData join.
- The score source extracts hundreds of fields even though this metric set consumes only a small fast-score subset.
- Passive-user and job-specific AppName/Scenario filters run after the score join.
- The Fusion View explicitly sets `@@ManagedOperatorChain = false`.
- `StripAllocationIds=true` from the job is overwritten by an unconditional false assignment in the Fusion View.
- The history path already narrows history rows before aggregation and is not a primary optimization target.

### Search QoS

- Date, entry-point, ring, and flight constraints are already pushed into the Logical Impression chain where supported.
- Request expansion is performed by an opaque custom processor before later deduplication and bot filtering.
- Deduplication windows by `UserId, ImpressionId`, but its hash hint contains only `UserId`.
- A production-path `ROWCOUNT=1000000000000` hint is far above observed job scale and can distort costing.
- Bot and test-traffic removal requires two large aggregations and two anti-semi joins.
- The job consumes 27 columns from the 59-column Metric View, but custom processors can limit column pruning.

## Unimplemented recommendations by priority

| Priority | Recommendation | Evidence / expected effect | Scope and risk | Why deferred / required validation |
|---|---|---|---|---|
| P0 | Push passive-user and AppName/Scenario filters before the BizChat score join | The slow merge processes wide usage and score rows before these filters. Earlier filtering should reduce join input, memory, and managed work. | Add explicit filter parameters to the source fusion module and apply them to UsageData. Medium compatibility risk. | The downloaded file is resolved output, and equivalent null/field mappings must be proven. Compile-plan and matched-job validation are required. |
| P0 | Add a BizChat fast-score projection mode | Hundreds of score fields cross the 4,000-way join while the metric set consumes a small subset. Narrowing the score payload should reduce I/O, serialization, and memory. | Change the source fusion module while preserving its default full schema. Medium implementation risk. | Requires downstream column-closure analysis, generated-plan inspection, and a matched job. |
| P0 | Enable managed operator chaining or fuse managed projections | Managed user code dominates hot stages, and the Fusion View explicitly disables chaining. | Remove the override or combine generated managed operators. High execution-path risk. | Requires compiler eligibility and plan validation before any run. |
| P1 | Fix `StripAllocationIds` and narrow requested flights earlier | The caller passes true, but the View immediately resets it to false. Correct handling may reduce unrelated flight rows before wide work. | Changes flight values and potentially row selection. High compatibility risk. | Requires value-level comparison of all flight formats and matched output. |
| P1 | Partition Search deduplication by `UserId, ImpressionId` | The window key has two columns, while the hint hashes only `UserId`; hot users can remain concentrated. | Change the partition key at the dedupe window. Medium physical-plan risk. | Output semantics are compatible, but skew, shuffle, and stage size must be measured. |
| P1 | Remove or replace the trillion-row Search hint | The hint is unsupported by observed scale and may cause excessive partitioning or conservative plans. | Remove it or use measured cardinality. Medium plan risk. | Requires compiled-plan comparison; no trustworthy replacement count is available locally. |
| P2 | Narrow Search custom-processor output | Only 27 of 59 Metric View columns reach the job, while the request extractor exposes a broad schema. | Add a projection mode or narrower processor contract. High maintenance risk. | Internal filters and helper calculations still consume some omitted columns; full liveness analysis is required. |
| P2 | Consolidate Search bot/test-traffic detection | Two large groupings and anti-semi joins run after request expansion. Shared pre-aggregation may reduce scans and shuffle. | Redesign filtering in the raw-log View. High semantic risk. | The two thresholds use different keys; equivalence needs data-level validation. |
| P3 | Investigate Nebula eligibility | Both chains use dynamic resolvers, custom processors, UDTs, and external assemblies. | Compiler/runtime migration. High risk. | Begin with compile-only eligibility; do not force fallback or alter outputs. |

No speculative percentage or duration improvement is assigned to these recommendations.

## Validation and limitations

- The two edited Views match frozen copies after normalizing only the intended edits.
- The other seven downloaded Views are byte-identical.
- View schemas and parameter blocks are unchanged.
- No new reference, resource, dependency, hint, rowset, or join was introduced.
- No script diagnostics were reported.

The assemblies, modules, resources, production data paths, and job-submission path are unavailable locally. Therefore,
the applied changes have static compatibility evidence but no measured runtime result.
