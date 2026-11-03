# Dust — Implementation Plan

**Project:** [rakmahefa/dust](https://github.com/rakmahefa/dust)  
**Plan date:** 2026-11-03  
**Baseline:** `main` at `76d79f25e4ea3ce355c4ac868f5ac9a233e64b89` (audit snapshot)  
**Related audits:** [files-audit.md](files-audit.md) · [status-audit.md](status-audit.md) · [performance-audit.md](performance-audit.md)  
**Status:** Proposed execution plan

---

## 1. Executive direction

Dust is a Rust Nintendo DS emulator with native desktop and WebAssembly frontends, a shared emulation core, software renderers and WGPU renderers.

The project already uses separate host threads for desktop emulation, selected rendering paths and audio handling. The next engineering cycle should therefore focus on **correctness, reproducible verification and measured performance**, not indiscriminate parallelization.

The first priority is to establish whether the threaded WGPU 3D rendering path is functional: at the audited revision, the call to `Renderer::render_frame` and queue submission is commented out in `render/wgpu-3d/src/threaded.rs`, despite a frame-rendering implementation existing in the crate. This is a correctness gate ahead of performance comparisons.

## 2. Objectives

1. Restore or explicitly gate incomplete rendering behavior, beginning with WGPU 3D.
2. Establish repeatable correctness checks for the core, renderers and frontends.
3. Make CI builds deterministic and diagnosable.
4. Add a benchmark and profiling baseline before changing hot paths.
5. Quantify synchronization, frame-copy and worker-handoff costs.
6. Preserve emulator timing, rendering correctness, audio continuity and browser compatibility during optimization.
7. Keep the fork maintainable and easy to synchronize with upstream.

## 3. Non-goals

The following are explicitly outside the first implementation cycle unless evidence changes the priority:

- A runtime CPU JIT. The optional JIT feature plumbing and generated tables do not constitute a completed runtime JIT; `core/src/cpu/jit.rs` is a TODO stub in the audited snapshot.
- Parallel execution of ARM7 and ARM9. The core scheduler advances both CPUs against shared emulated time, shared buses and scheduled device events. Do not split them across host threads without a separate correctness design.
- Large-scale renderer rewrites or replacement of the existing triple-buffer and ownership models without profiling and concurrency review.
- Claims of performance improvement based only on average FPS.
- Automatic adoption of AppImage, Flatpak or another Linux packaging format without defining distribution targets and validating the result.

## 4. Guiding engineering principles

- **Correctness before speed:** validate rendering and emulator behavior before using a backend as a performance baseline.
- **Measure before optimizing:** every performance change must identify a workload, metric and reproducible before/after result.
- **Preserve hardware timing:** do not change CPU/device scheduling semantics to simplify parallel execution.
- **Bound concurrency:** keep worker ownership explicit; avoid unbounded queues and accidental extra copies.
- **Use deterministic builds:** rely on committed lockfiles and pinned toolchain inputs where practical.
- **Keep PRs narrow:** one specialist workstream per PR; no broad drive-by refactoring.
- **Document uncertainty:** distinguish confirmed source behavior from runtime observations and hypotheses.

## 5. Prioritized workstreams

| ID | Priority | Workstream | Primary ownership | Dependencies |
| --- | --- | --- | --- | --- |
| W1 | P0 | WGPU 3D rendering correctness and submission | WGPU rendering specialist | None |
| W2 | P1 | Core correctness tests and smoke-test strategy | Emulator QA specialist | Can start with W1 |
| W3 | P1 | CI reproducibility and build matrix hardening | CI/build specialist | W2 test command agreement |
| W4 | P1 | Native runtime profiling and benchmark baseline | Performance specialist | W1 backend correctness |
| W5 | P2 | WebAssembly frame-transfer and worker performance | WebAssembly specialist | Browser correctness baseline |

These workstreams may be investigated in parallel. Their merge order must respect dependencies and any overlapping code changes.

## 6. Delivery phases

### Phase 0 — Rendering correctness gate

**Goal:** establish a known-good WGPU 3D path or make its incomplete status explicit.

Scope:
- Trace the flow from GX/frame preparation through the WGPU renderer worker.
- Verify triple-buffer ownership, frame indices, worker wake-up, command encoder creation, submission and output-view synchronization.
- Determine why the render call is commented out and whether the referenced upstream API constraint is still applicable.
- Exercise a deterministic 3D test workload on supported hardware and at least one fallback adapter where possible.
- Verify software 3D and WGPU 3D output against a known reference.

Exit criteria:
- The backend visibly renders the intended frame and submits the command buffer, or is explicitly gated/disabled with an actionable diagnostic.
- No deadlock, stale-frame loop, output-view race or incorrect frame ownership is introduced.
- Test instructions and hardware/backend limitations are recorded.

### Phase 1 — Correctness and verification foundation

**Goal:** make regressions detectable without relying on manual game testing alone.

Scope:
- Inventory existing inline tests and identify critical untested core behavior.
- Add focused deterministic unit tests for selected scheduler, CPU, memory, GPU and savestate invariants.
- Define a smoke-test ROM strategy based on redistributable homebrew/test ROMs; do not commit copyrighted commercial ROMs.
- Add test documentation including expected output, prerequisites and known limitations.
- Ensure debug and release configurations are both considered where behavior differs.

Exit criteria:
- A documented local test command works from a clean checkout.
- Tests are deterministic and independent of proprietary ROMs.
- Failures provide enough context to identify the affected subsystem.

### Phase 2 — CI and build reproducibility

**Goal:** make pull requests and release builds trustworthy and repeatable.

Scope:
- Remove routine `cargo update` from CI build and lint jobs; use `--locked` where compatible.
- Add a committed npm lockfile and use `npm ci` for web builds.
- Add explicit formatting and test checks.
- Keep native target/feature coverage for Windows, Linux and macOS.
- Verify WebAssembly build prerequisites and cross-origin isolation headers.
- Separate PR validation from deployment so pull requests do not publish production artifacts.
- Evaluate Linux packaging formats against intended users; retain the existing archive unless AppImage requirements and tests justify a new artifact.

Exit criteria:
- CI uses locked dependency resolution.
- Formatting and test failures block merges.
- Build artifacts are uploaded only after successful packaging and validated for expected files.
- Web deployment is restricted to the intended branch/event and has a reproducible build.

### Phase 3 — Native profiling and measured optimization

**Goal:** identify the largest user-visible costs before changing runtime behavior.

Areas to measure:
- Emulation thread time per frame and per CPU core.
- ARM instruction dispatch, bus/memory fallback paths and device scheduling.
- Software 2D scanline worker waits and renderer CPU time.
- WGPU worker handoff, frame submission and capture waits.
- Desktop frame limiter behavior while active and paused.
- Framebuffer copies, allocations and frame pacing variance.
- Audio buffer occupancy, synchronization waits and underruns.
- x86_64 AVX2 versus general software 2D rendering.

Exit criteria:
- A baseline report records hardware, OS, compiler/toolchain, commit, feature set, renderer, workload, warm-up and sampling duration.
- Median and p95 frame times are recorded, alongside CPU/GPU utilization and audio stability where available.
- Each proposed optimization names the measured bottleneck and expected trade-off.

### Phase 4 — WebAssembly and frontend performance

**Goal:** determine whether WebAssembly frame allocation/copying or worker messaging is material.

Scope:
- Measure the cost of `Uint32Array::from` in `frontend/web/crate/src/lib.rs::run_frame`.
- Measure worker message time and transferred buffer lifetime.
- Prototype reusable buffers or shared-memory exchange only if the copy is a significant part of frame time.
- Preserve explicit ownership and synchronization between emulator worker, renderer worker and UI.
- Verify browser requirements for shared memory, including COOP/COEP, and provide an understandable fallback when unsupported.

Exit criteria:
- A repeatable browser workload produces transfer/allocation timings.
- Any new buffer protocol defines producer, consumer, reuse point and back-pressure.
- No partially rendered frame, data race or buffer reuse before consumption is possible.
- Supported browser requirements and fallback behavior are documented.

## 7. Cross-cutting acceptance criteria

Every implementation PR must:

1. State the baseline commit and exact files/components changed.
2. Include a concise rationale and the observable problem it solves.
3. Include tests or a reproducible manual validation procedure.
4. Run formatting, relevant tests, Clippy where applicable and the target build.
5. Report failed or unavailable checks honestly.
6. Avoid unrelated refactors and dependency upgrades.
7. Explain concurrency and unsafe-memory assumptions where applicable.
8. Include before/after measurements for performance claims.
9. Preserve software-renderer or other fallback behavior when a feature is unavailable.

Performance changes are accepted only when repeated measurements on the same machine/workload show a meaningful improvement in a named metric without correctness, frame-pacing, audio or memory regressions.

## 8. Risks and controls

| Risk | Impact | Control |
| --- | --- | --- |
| WGPU submission API or lifecycle is misunderstood | Black screen, stale frames or worker deadlock | Trace full render lifecycle and validate on a known 3D workload |
| Unsafe shared buffers are modified without ownership proof | Data races or corrupted frames | Document ownership; review all unsafe accesses and synchronization transitions |
| ARM7/ARM9 execution is parallelized prematurely | Timing and device-order regressions | Keep core execution sequential unless a dedicated design proves determinism |
| Busy-wait removal increases latency | Frame pacing or capture regressions | Compare adaptive/blocking wait against baseline using both latency and CPU metrics |
| Dependency updates change CI behavior | Non-reproducible builds | Lock Rust and Node dependency resolution |
| Wasm shared memory is unavailable in a browser context | Startup failure or fallback incompatibility | Check cross-origin isolation and document supported browser behavior |
| Benchmark workload is not representative | Optimizations target the wrong bottleneck | Use multiple 2D, 3D, audio and idle workloads |

## 9. Recommended merge sequence

1. W1: WGPU 3D correctness PR.
2. W2: core tests and smoke-test documentation.
3. W3: CI reproducibility and verification jobs.
4. W4: native profiling and benchmark tooling.
5. W5: WebAssembly measurements and any justified buffer optimization.

The workstreams can investigate in parallel, but merge and integration should follow this sequence where code or test dependencies overlap.

## 10. Relationship to existing project TODO

The upstream `TODO.md` records hardware research, missing emulator functionality and frontend ideas. It must remain intact. This plan governs the next engineering cycle and does not imply that all existing hardware TODOs are completed or superseded.

The execution checklist is maintained separately in [execution-todo.md](execution-todo.md).
