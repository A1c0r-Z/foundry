# NOTICE

This repository is a fork of upstream Foundry
(https://github.com/foundry-org/foundry) at tag `v0.0.1b` (commit `118081a`),
distributed under the Apache License 2.0 (see LICENSE).

## Local modifications on top of upstream `118081a`

The following files are modified relative to the upstream `118081a` snapshot.
Total: **349 LOC across 5 files**, written **2026-05-15** by ce.zhao
(Zoom Communications) as independent work.

```
csrc/CUDAGraph.cpp         +11
csrc/CUDAGraphParallel.cpp +5
csrc/binding.cpp           +64
csrc/hook.cpp              +295
include/hook.h             +10
```

### Purpose

Upstream Foundry `118081a` ships an internally-implemented passthrough
recording mechanism — `start_hook_record()`, `end_hook_record()`,
`HookAllocationEvent`, `hook_recording_enabled`, `hook_alloc_events`, and
recording calls embedded in `cuMemAlloc_v2` — but **does not expose any
of it through `binding.cpp`**. Downstream Python clients that try to
import these names fail with `AttributeError`.

This patchset exposes the pre-existing internal machinery as a Python
API surface and adds the bits required for cross-process replay.

### What was added

- `binding.cpp` (+64): Python bindings for `start_passthrough_record`,
  `end_passthrough_record`, `pause_passthrough_record`,
  `resume_passthrough_record`, `get_passthrough_events`,
  `replay_passthrough_events`. The first five are thin wrappers over
  upstream's already-implemented `start/end/pause/resume_hook_record`
  internals (see [`hook.cpp:3110-3115`](csrc/hook.cpp#L3110) and
  [`include/hook.h:40-41`](include/hook.h#L40) in pristine `118081a`).
  `replay_passthrough_events` is new — it drives a new internal
  `replay_passthrough_events_internal` helper to reproduce a save-time
  alloc sequence at load time.

- `hook.cpp` (+295):
  - `hook_fallback_events` — separate vector (pristine has none) capturing
    `cuMemCreate` / `cuMemMap` / `cuMemSetAccess` failure-fallback
    allocations, which produce driver-chosen VAs that PyTorch's caching
    allocator later carves tensors out of. Must survive across the
    `hook_alloc_events.clear()` calls that pristine performs on graph
    capture end.
  - `hook_capture_start_index` marker so `save_hook_events_to_json` only
    emits events from the current capture window (pristine emits all
    accumulated events, leaking prior windows).
  - Passthrough-record path additions inside `cuMemAlloc_v2`,
    `cuMemAllocPitch_v2`, `cuMemAddressReserve` for the
    bump-region-disabled code path (pristine only records inside the
    bump path).
  - `replay_passthrough_events_internal` — sequential replay of save-time
    events through `cuMemAlloc_v2`, advancing the load process's bump
    cursor to match save-time positions.
  - `[ALLOC_TRACE]` diagnostic prints.

- `CUDAGraph.cpp` (+11), `CUDAGraphParallel.cpp` (+5),
  `include/hook.h` (+10): supporting declarations and minor plumbing
  changes consistent with the additions above.

### Relationship to lokashrinav's fork

The lokashrinav fork's commit
[`a7d97e4`](https://github.com/lokashrinav/foundry/commit/a7d97e4)
("Add passthrough recording, multi-device access, and IPC support"),
authored **2026-05-18**, addresses the same exposure-gap problem
**independently and three days later** than this patchset (2026-05-15).
The two implementations are not related:

- This patchset calls upstream's pre-existing `start_hook_record()` /
  `end_hook_record()` internals.
- `a7d97e4` instead introduces *new* internal functions named
  `start_passthrough_record()` / `end_passthrough_record()` and calls
  those.

Line-by-line `binding.cpp` overlap between the two is near-zero
(46 lines unique to ours, 24 unique to theirs, no shared implementation
lines). The patchsets were written without knowledge of one another.

### License

All modifications are released under the original Apache License 2.0.
See LICENSE for full terms.
