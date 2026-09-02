# Foray Optimization Summary

## Scope

This report summarizes the optimization work completed for:

- `12c543ef-123d-44f5-9ee6-f63b9fe05f27.script` (Search QoS)
- `1ab1998c-bbff-4c2e-b54e-f8994942d2bc.script` (BizChat)
- Their downloaded Search, BizChat, flight, score, and Logical Impression Views

Output metrics, segments, schemas, paths, filters, flight rules, reducer keys, histograms, and expiry settings were
treated as compatibility requirements.

## Applied optimizations

### Generated scripts

- Removed dead Search projections and redundant generated boolean/status expressions.
- Replaced 595 constant Search `ToLower()` calls and 62 BizChat lowercase allocations.
- Reduced BizChat JSON parsing, including direct `R9Latency` parsing from 59 to 13 call sites.
- Materialized two BizChat latency sequences to prevent three enumerations of each sequence.
- Changed the BizChat Dataverse maximum-latency calculation from two array passes to one.
- Removed a redundant Search empty-flight regex and two duplicate `StandardizedFlightParser` calls.
- Combined separate Search null and empty `RandomizationId` checks.
- Removed two ignored, non-required Search partition hints while retaining required production hints.
- Preserved every consumed Native, Bootstrap, Histogram, metric, and segment output.

### Downloaded Views

- Moved the BizChat score `RequestId` filter before wide normalization and five `SSRMetrics` JSON extractions.
- Removed the unused BizChat `ScoreDataNotToJoin` rowset.
- Replaced the impossible BizChat `IsOnlyScoreData` condition with constant `false`.
- Changed the Search abandoned-session test from `Where(...).Count() > 0` to short-circuiting `Any(...)`.
- Avoided lowercase allocation in that Search action-type comparison.

The edited Copilot Data Fusion View is a resolved job artifact. Its changes should ultimately be made in the source
module that generates the resolved View.

## Kusto investigation

The complete telemetry pass used different stores for the two jobs:

- Search QoS: West US `events-cosmos`
- BizChat: North Europe `events-synapsescope`

Neither store exposes a dedicated `JobGraph` table. Critical paths were reconstructed from `JobOperators`,
`CriticalPathVertexEvent`, `CompilationStage`, vertex telemetry, and the downloaded View dependencies.

### Search QoS

- The job ran for 10 hours 11 minutes and failed with
  `E_USER_JM_AM_TempDataHeldBytesPerTokenLimitExceeded`.
- It completed 241,797 logical vertices across 347 stages and cumulatively wrote 501.24 TiB.
- `SV339_Aggregate_1_Agg_Internal` had a 14.25 input-skew ratio and wrote 134,552.60 GiB.
- `SV339_Aggregate` sorted 2.49 billion rows and wrote 101.35 TiB through 35 to 37 spill buckets.
- The completed outer partitions were balanced, but the internal fan-in was highly bimodal.
- Compilation estimated about 2,981 bytes per final-stage row; operator telemetry measured about 44,703 bytes.
- The reconstructed terminal path is:
  `SV273_Dataset -> SV339_Aggregate -> SV347_Combine`.

The immediate problem is final metric aggregation and temp-data retention, not source-extract partition count alone.
Changing reducer fan-in, partitioning, row estimates, or metric aggregation shape would change the physical plan and
cannot be validated by static output comparison.

### BizChat

- The job succeeded in 18 hours 58 minutes of execution with 123,956 completed logical vertices.
- `SV217_Combine_Split` and `SV225_Aggregate` consumed 71.5% of accumulated active vertex time.
- `SV217_Combine_Split` is a 4,000-way merge join over 540.1 million input rows and 48,443.69 GiB.
- Join input averaged about 96 KB per row; read skew was only 1.01.
- Managed user code consumed about 6,926 exclusive hours in the join stage.
- Fifty `TASK_MemoryEstimateTooLow` attempts affected 47 vertices. Failed attempts reached up to 5.99 GiB.
- Successful retries used containers up to 12 GiB, while the initial allocation was 3 GiB.
- `SV225_Aggregate` expanded 475.1 million rows into 9.50 billion managed row invocations before later reduction.
- The principal critical path is:
  `SV214 + SV216 -> SV217 -> SV223 -> SV224 -> SV225 -> SV226 -> SV227 -> SV228 -> SV230 -> SV231`.

The evidence confirms that wide join payload, repeated managed processing, late cardinality reduction, and memory
underestimation are the main BizChat costs. Adding partitions would not address the measured bottleneck.

No additional script or View edit met both requirements of exact output equivalence and no unvalidated physical-plan
change. The earlier safe edits remain applicable and are listed above.

## Unimplemented opportunities

| Priority | Opportunity | Reason deferred |
|---|---|---|
| P0 | Reduce Search final-aggregation temp data | Requires a new reducer/aggregation physical plan. |
| P0 | Correct Search final-stage byte estimates | Requires compiler statistics or measured cardinality input. |
| P0 | Push BizChat filters before the score join | Needs source parameters and matched output validation. |
| P0 | Narrow fast-score data before the BizChat join | Needs column and plan validation. |
| P0 | Chain or fuse managed BizChat processors | Changes execution boundaries. |
| P1 | Fix `StripAllocationIds` and narrow flights | Can change flight values and selected rows. |
| P1 | Correct BizChat hot-stage memory sizing | This is a scheduler/VSC change, not a safe script rewrite. |
| P1 | Partition Search deduplication by `UserId, ImpressionId` | Changes shuffle and partition behavior. |
| P1 | Remove or replace the Search trillion-row estimate | No reliable replacement cardinality is available locally. |
| P1 | Tune Search final aggregation metadata | Needs cardinality, skew, and plan evidence. |
| P2 | Narrow Search processor output | Opaque processors may need apparently unused columns. |
| P2 | Consolidate Search traffic aggregations | Different keys require data-level proof. |
| P2 | Share repeated BizChat segmentation work | A shared rowset could increase row width and memory. |
| P2 | Reassess BizChat `LOWDISTINCTNESS` hints | Actual distinct counts are required. |
| P3 | Investigate SSS flags and Nebula | Custom managed components make this high risk. |

## Result and limitation

The completed work removes repeated parsing, enumeration, string allocation, dead expressions, and known ignored
hints without intentionally changing outputs. The larger remaining opportunities affect source View interfaces,
physical plans, joins, partitions, or scheduler behavior.

The jobs cannot currently be submitted, so no measured runtime improvement is claimed. Plan-changing recommendations
should be evaluated with compile-plan comparison and matched job telemetry before production use.
