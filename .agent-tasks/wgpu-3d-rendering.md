# Agent handoff

**Specialist:** `wgpu-rendering-specialist`  
**Target branch:** `agents/wgpu-3d-rendering`  
**Base:** `main`  
**Task:** P0 — Verify and restore WGPU 3D frame submission

# Assigned work: WGPU 3D rendering correctness

Use repository custom agent **`wgpu-rendering-specialist`**.

## Context
The implementation plan ranks WGPU 3D correctness as P0. At the audit baseline, `render/wgpu-3d/src/threaded.rs` comments out the call to `renderer.render_frame(&frame.rendering_data)` and `renderer.queue().submit(...)` behind a FIXME. A render implementation exists in `render/wgpu-3d/src/lib.rs`.

## Required work
1. Trace the full render lifecycle: GX state preparation, triple buffer, frame index, worker wake-up/park, render encoding, submission and output-view publication.
2. Verify the current WGPU 23 API and investigate the original FIXME; do not uncomment the code without understanding the lifecycle/API issue.
3. Restore a valid command-buffer submission and synchronization path if possible. Otherwise gate the backend clearly and report the blocker.
4. Validate normal frames, capture synchronization and renderer shutdown.
5. Add focused regression coverage or provide an exact reproducible test procedure using a redistributable homebrew/test ROM.
6. Keep changes scoped to WGPU rendering and directly necessary desktop renderer integration.

## Acceptance criteria
- 3D output is rendered/submitted and displayed on a known test workload, or the backend clearly reports that it is unavailable.
- No stale-frame loop, deadlock, incorrect frame ownership or capture regression.
- Report commands, hardware/adapter/backend, workload and test results.

Do not claim runtime success unless actually observed.

## Repository guardrails

- Read `implementation-plan.md`, `execution-todo.md` and the relevant audits first.
- Work only within the assigned scope.
- Run the narrowest relevant tests, formatting and build checks.
- Do not merge the PR. Request review and report any blockers.
- Update this handoff with files changed, commands executed, results and known limitations.
