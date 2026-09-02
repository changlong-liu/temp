# Foray Script Performance Optimization Report

## Scope

This optimization pass updated:

- `12c543ef-123d-44f5-9ee6-f63b9fe05f27.script`
- `1ab1998c-bbff-4c2e-b54e-f8994942d2bc.script`

All output metrics, segments, paths, headers, unions, histograms, reducer keys, and expiry settings were preserved.

## Changes

### Search QoS script

- Replaced 595 constant `ToLower()` calls with lowercase literals.
- Simplified 150 constant HTTP status branches.
- Removed 208 redundant `&& true` expressions.
- Removed 12 redundant `== true` checks.
- Kept the timestamp conversion and fixed partition counts unchanged because they require execution-level evidence
  before they can be changed safely.

### BizChat script

- Removed the standalone `FilteredInput` rowset and restored the same View and filters directly under `ROWSET1`.
- Reduced repeated parsing within individual expressions.
- Replaced 62 lowercase-string allocations with ordinal case-insensitive comparisons.
- Folded 27 constant expressions and two redundant empty checks.
- Kept all consumed Bootstrap and Histogram calculations.

## Results

| Measurement | Before | After | Reduction |
|---|---:|---:|---:|
| Search script size | 304,261 bytes | 281,525 bytes | 22,736 bytes |
| Search `ToLower()` calls | 596 | 1 | 595 |
| Search constant status branches | 150 | 0 | 150 |
| BizChat script size | 2,469,604 bytes | 2,466,387 bytes | 3,217 bytes |
| BizChat `JObject.Parse` calls | 107 | 103 | 4 |
| BizChat `JArray.Parse` calls | 156 | 150 | 6 |
| BizChat `ToLower()` calls | 62 | 0 | 62 |
| BizChat `FilteredInput` rowsets | 1 | 0 | 1 |

Compared with the original generated files, the cumulative parsing counts in the BizChat script changed as follows:

- `JObject.Parse`: 175 to 103.
- `JArray.Parse`: 160 to 150.

## Compatibility

Static comparisons found no changes to:

- Metric lists or ordering.
- Segment and profile definitions.
- Output paths and metadata headers.
- Native, Bootstrap, and Histogram unions.
- Histogram expiry.
- Source View parameters and predicates.
- Search metric-dimension mappings, partitions, and reducer keys.

Both optimized scripts have no IDE diagnostics.

## Limitations

The directory does not include the referenced runtime assemblies, resources, or production View access, so the scripts
could not be compiled or executed locally. Runtime improvements must be confirmed using matched job telemetry.

The following changes were intentionally not made:

- Cross-expression or cross-row JSON caching.
- Removal of consumed metrics, segments, Bootstrap statistics, or histograms.
- Timestamp semantic changes.
- Partition-count changes without data-volume and skew measurements.

Direct edits can be overwritten if the scripts are regenerated.
