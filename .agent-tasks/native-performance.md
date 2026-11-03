# Agent handoff

**Specialist:** `performance-engineering-specialist`  
**Target branch:** `agents/native-performance`  
**Base:** `main`  
**Task:** P1 — Instrument native performance and establish benchmarks

# Assigned work: native profiling and performance baseline

Use repository custom agent **`performance-engineering-specialist`**.

## Context
The audit identified measurable candidates, not proven bottlenecks: software 2D scanline busy waits, desktop frame-limit spin, WGPU worker/capture handoffs, audio synchronization, framebuffer copies and x86_64 AVX2 implementations. ARM7 and ARM9 share emulated time and device scheduling.

## Required work
1. Add opt-in timing/profiling instrumentation that does not alter default emulator timing.
2. Measure emulation and renderer time per frame.
3. Measure scanline waits, desktop spin window, WGPU handoff/capture waits (only after WGPU path is validated) and audio waits separately.
4. Verify actual AVX2 runtime dispatch and compare AVX2/general paths on identical hardware/workloads.
5. Profile instruction dispatch and bus/memory fallback before proposing a JIT or parallel CPU execution.
6. Run repeatable warmed workloads, reporting median/p95 frame time, CPU usage, pacing and audio underruns where available.
7. Do not change hot-path synchronization or CPU scheduling based only on source inspection. Do not implement speculative optimizations without measured evidence.

## Acceptance criteria
- Reproducible native baseline report with machine, OS, compiler, commit, features, renderer and workload.
- Profiling is opt-in and has no material impact when disabled.
- Recommendations identify measured cost and trade-offs; no unsupported performance claims.

## Repository guardrails

- Read `implementation-plan.md`, `execution-todo.md` and the relevant audits first.
- Work only within the assigned scope.
- Run the narrowest relevant tests, formatting and build checks.
- Do not merge the PR. Request review and report any blockers.
- Update this handoff with files changed, commands executed, results and known limitations.
