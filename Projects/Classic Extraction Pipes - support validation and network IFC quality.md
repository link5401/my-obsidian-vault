---
categories:
  - "[[Projects]]"
tags:
  - issues
  - quality
  - ifc
  - networks
  - pipe-extraction
type:
  - project
status: active
start: 2026-05-13
topics:
  - "[[classic-extraction-pipes]]"
  - "[[support validation]]"
  - "[[network IFC]]"
  - "[[dataset-004]]"
---

## Summary

- Goal: improve final `pipe_network.ifc` quality for `dataset-004`.
- Current command focus: `--no-support-normal-check`, `--no-support-connected-component-check`, and `--support-validate-multisegment`.
- Current finding: support validation matters, but dedupe is also still too aggressive and can remove good geometry before or during topology/export preparation.

## Current Status

- `--support-validate-multisegment` is still the only one of the three support flags actively adding protection in the current command.
- `--no-support-normal-check` and `--no-support-connected-component-check` both bias the run toward higher recall.
- The attempted obs-count-based dedupe fix was directionally useful, but the dedupe behavior is still too aggressive.

## Export Impact

These checks run before topology reconstruction, so they affect geometry-derived exports much more than raw point cloud exports.

- Most affected: `pipe_network.ifc`, `pipes.ifc`, `pipes.ifc.json`, `simulated.ply`
- Not directly affected: `detections.e57`, `pipe_only_tsdf.ply`, `detections_tsdf_colored.ply`

Why `pipe_network.ifc` is most sensitive:

- Any false accept, false reject, or over-merge changes which source geometries survive into topology reconstruction.
- That changes junction placement, run-edge creation, and final export provenance.

## Support Flag Notes

### `--no-support-normal-check`

- Disables the surface-normal consistency test during geometry support validation.
- If enabled, failing segments reject with `low_normal_consistency`.
- Tradeoff: helps recall when normals are noisy, but can keep physically implausible geometry alive.

### `--no-support-connected-component-check`

- Disables the connected-support test for inlier points.
- If enabled, failing segments reject with `fragmented_support`.
- Tradeoff: helps when support is broken by occlusion or depth gaps, but can allow disjoint evidence to be treated as one pipe.

### `--support-validate-multisegment`

- Keeps support validation active for bent or piecewise hypotheses.
- If disabled, multi-segment hypotheses bypass this stage with `multisegment_bypass`.
- Tradeoff: protects network quality by validating bends, but can reject real multi-segment pipes when support is sparse.

## Command Reading

- The current command is still recall-leaning overall.
- If the goal is specifically a cleaner `pipe_network.ifc`, the first support switch worth testing back on is the connected-component check.
- The normal check is likely more dataset-sensitive and depends on normal quality.

## Attempt Log

### 2026-05-13: Obs-count-based dedupe fix

Problem observed:

- `_dedupe_geometries` in `pipeline.py` preferred the longest parallel instance.
- That let `inst=76` (12.8 m, obs=60) absorb `inst=70` (12.7 m, obs=171) before topology construction.
- Only 22 of 40 accepted instances reached `build_pipe_network`.

Implementation attempted:

- `geometry_extractor.py`
  Added `obs_count: int = 0` to `PipeGeometryHypothesis`.
- `geometry_extractor.py`
  Populated `obs_count` from `instance.observation_count` at both extraction call sites.
- `pipeline.py`
  Changed `_dedupe_geometries` sorting to `(obs_count desc, length desc)` so the most-observed instance wins.
- `pipeline.py`
  Derived `obs_count_map` from geometries and passed it to both `simplify_network_for_export` calls.
- `topology_reconstructor.py`
  Updated `_dedupe_parallel_edges` to accept `obs_count_map`.
- `topology_reconstructor.py`
  Sorted parallel groups by `(obs_count, length) desc`.
- `topology_reconstructor.py`
  Merged all group source IDs into the winner with the highest-observation source ID first so `_primary_export_source_ids` prefers the most-observed instance.
- `topology_reconstructor.py`
  Threaded `obs_count_map` through `simplify_network_for_export`.

Intended effect:

- Preserve the best-supported geometry when near-duplicate candidates compete.
- Reduce premature loss of strong instances before topology reconstruction.
- Improve `pipe_network.ifc` source selection and export provenance.

Result so far:

- Attempt recorded.
- Dedupe is still too aggressive.
- The issue is not fully resolved by switching the winner from longest to most-observed.

## Next Check

- Inspect where remaining geometry loss happens after the winner-selection change.
- Determine whether the remaining over-merge is happening in `_dedupe_geometries`, parallel-edge dedupe, or later topology simplification.
- Compare source coverage and final `pipe_network.ifc` quality on `dataset-004` after this attempted fix, with attention to whether valid parallel runs are still being collapsed.
