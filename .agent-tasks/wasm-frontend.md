# Agent handoff

**Specialist:** `wasm-frontend-specialist`  
**Target branch:** `agents/wasm-frontend`  
**Base:** `main`  
**Task:** P2 — Measure and optimize WebAssembly frame transfer

# Assigned work: WebAssembly frontend and frame-transfer performance

Use repository custom agent **`wasm-frontend-specialist`**.

## Context
The audit found that `frontend/web/crate/src/lib.rs::run_frame` creates a `Uint32Array` from the framebuffer slice for each frame. The emulator runs in a worker, with a separate 3D renderer worker. Wasm atomics/shared memory are configured and deployment headers set COOP/COEP.

## Required work
1. Map UI, emulator worker and renderer worker message/ownership flow.
2. Measure framebuffer conversion/allocation, worker transfer latency, frame pacing and allocation volume in a supported browser.
3. Confirm the actual browser requirements for Wasm shared memory and cross-origin isolation; do not remove COOP/COEP headers as a shortcut.
4. Determine whether the copy is material to frame time.
5. Only if measurement justifies it, prototype reusable or shared-memory buffers with explicit producer/consumer ownership and back-pressure.
6. Validate start/stop/reset/pause/resume, tab visibility and repeated game loads.
7. Preserve a safe fallback and document unsupported browser behavior.

## Acceptance criteria
- Browser and workload are identified; timings are reproducible.
- Any proposed buffer protocol defines ownership and exact reuse points.
- No stale/partial frames, buffer reuse races, unbounded queues or audio regressions.
- Report measured before/after results; do not claim zero-copy unless demonstrated.

## Repository guardrails

- Read `implementation-plan.md`, `execution-todo.md` and the relevant audits first.
- Work only within the assigned scope.
- Run the narrowest relevant tests, formatting and build checks.
- Do not merge the PR. Request review and report any blockers.
- Update this handoff with files changed, commands executed, results and known limitations.
