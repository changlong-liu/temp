# Foray SCOPE Physical-Plan Optimization Report

## Result

This pass made two output-compatible changes:

- Search QoS: removed two non-required `PARTITIONCOUNT = 4000` hints that were ignored at their current statement
  boundaries.
- BizChat: materialized two filtered latency sequences once before `Count()`, `Sum()`, and `Count()` consume them.

No metric, segment, schema, output path, header, reducer key, source predicate, histogram output, or expiry setting was
changed.

## Applied changes

| Script | Change | Before | After |
|---|---|---:|---:|
| Search QoS | Non-required 4,000-partition hints | 2 | 0 |
| Search QoS | Required production round-robin hint | 1 | 1 |
| BizChat | Materialized latency sequences | 0 | 2 |
| BizChat | `JObject.Parse` call sites | 103 | 103 |
| BizChat | `JArray.Parse` call sites | 150 | 150 |
| BizChat | `LOWDISTINCTNESS` hints | 65 | 65 |
| BizChat | Required partition hints | 7 | 7 |

The BizChat change preserves the original positive-latency filter, nullable result, arithmetic average, and malformed
input behavior. It only prevents the lazy sequence, including its JSON-object conversion, from being enumerated three
times.

## Existing-job observations

- Search QoS was dominated near the final aggregation. `SV339_Aggregate` used 1,519 vertices at about 68.5 GB average
  input per vertex, and its internal aggregate showed an input-skew ratio of about 14.2.
- Many Search source-View extract stages already used 1,000 partitions. Wrapper-level partition hints cannot correct
  all upstream View expansion and skew.
- BizChat `SV217_Combine_Split` used 4,000 vertices and read two 4,000-stream inputs. Row distribution was relatively
  uniform, while managed user code dominated operator time.
- `SV217_Combine_Split` also had 50 `TASK_MemoryEstimateTooLow` attempts. This indicates a resource-estimation problem,
  not evidence that more partitions or more `LOWDISTINCTNESS` hints are needed.
- BizChat already has 65 `LOWDISTINCTNESS` hints and seven required 4,000-partition hints. Adding more without a
  matched run could increase shuffle, fan-in, or optimizer estimation error.

## Unimplemented recommendations by priority

| Priority | Recommendation | Evidence and expected effect | Why not implemented / required validation |
|---|---|---|---|
| P0 | Push filtering and payload trimming into the BizChat source View | The hot stage is a 4,000-way merge join over very wide rows. Earlier filtering and narrower join payloads can reduce read, memory, and per-row managed work. | The View definition is not available with these scripts. Its filter semantics and required join columns must be verified before changing it. |
| P0 | Enable managed operator chaining or fuse adjacent generated C# processors | Managed user code dominates hot operators. The script invokes 12 opaque `CSharpExpressionProcessor` operators, including adjacent layers in turn-level paths. Fewer boundaries can reduce interop and intermediate-row overhead. | This changes generated code and physical execution. It requires at least compiler-plan validation and normally a matched job. |
| P1 | Correct the BizChat `SV217` VSC memory estimate | There were 50 low-memory retries at about 6 GB peak memory before higher-resource attempts succeeded. Correct sizing should reduce retries and wasted attempts. | This is a scheduler/resource configuration change rather than a safe script rewrite. |
| P1 | Place the Search partition request at the final reducer boundary | The hot aggregate used 1,519 large vertices, while the current PROCESS-level hints were ignored. A reducer-level request is more likely to affect the intended shuffle. | `REQUIRED=true` materially changes the plan. Compile and run validation are required before applying it. |
| P1 | Add targeted Search low-cardinality metadata | `FLIGHT`, `Segment_Name_1`, and `Segment_Name_2` are plausible low-cardinality keys around the skewed final aggregation. Accurate metadata may improve aggregation planning. | Cardinality hints can worsen a plan when assumptions are wrong. Cardinalities and a matched run are required. |
| P2 | Factor the six identical BizChat segmentation passes | Six branches use the same `SegmentProcessorV2` configuration. Sharing segment expansion could remove repeated processor work. | A shared rowset would carry the union of 27-to-254-column branch projections and may increase row width and memory. |
| P2 | Remove high-cardinality columns from BizChat `LOWDISTINCTNESS` hints | Seventeen hints include `SESSION_ID`; 64 include segment-value columns. Treating high-cardinality values as low-cardinality can distort estimates. | Actual distinct counts and plan comparison are required; changing metadata without them is speculative. |
| P3 | Investigate SSS optimizer flags | The Search job already set a very high byte limit. Its warnings reported absent streamsets, not byte or stream-count limits. | `MaxStreamsPerSSSExtractCombineVertex` does not match the observed warning and should not be guessed. |
| P3 | Investigate Nebula eligibility | Both scripts use custom processors, reducers, statistics UDTs, external assemblies, and inline managed expressions. | No Nebula setting appeared in the jobs, and the custom managed surface is likely ineligible. Start with compile-only eligibility checking. |

## Validation and limitation

The optimized files were compared with frozen pre-change copies after normalizing only the four intended edits. All
remaining content was byte-identical, and no script diagnostics were reported.

Job submission was not available, so this report does not claim measured runtime improvement. The P0-P3 items remain
recommendations rather than applied changes.
