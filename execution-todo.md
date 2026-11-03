# Dust — Execution TODO

**Plan:** [implementation-plan.md](implementation-plan.md)  
**Audits:** [files-audit.md](files-audit.md) · [status-audit.md](status-audit.md) · [performance-audit.md](performance-audit.md)  
**Baseline under audit:** `76d79f25e4ea3ce355c4ac868f5ac9a233e64b89`  
**Last updated:** 2026-11-03

This checklist tracks the next implementation cycle. The upstream [TODO.md](TODO.md) is preserved as the long-running hardware research backlog.

## Status legend

- [ ] Not started
- [~] In progress
- [x] Completed and verified
- [!] Blocked or requires a decision

## Parallel agent dispatch — 2026-11-03

The repository has GitHub Issues disabled, so the specialist tasks are tracked as isolated draft pull requests. A `@copilot` dispatch comment targeting each custom profile has been posted. **Dispatch requests are sent; Copilot acceptance/activity has not yet been observed in the PR conversations at the time of this update.**

| Workstream | Custom agent | Branch | Draft PR | Dispatch status |
| --- | --- | --- | --- | --- |
| W1 — WGPU 3D correctness | `wgpu-rendering-specialist` | `agents/wgpu-3d-rendering` | [#1](https://github.com/rakmahefa/dust/pull/1) | Comment sent; awaiting agent activity |
| W2 — Emulator QA/tests | `emulator-qa-specialist` | `agents/emulator-qa` | [#2](https://github.com/rakmahefa/dust/pull/2) | Comment sent; awaiting agent activity |
| W3 — CI/build reproducibility | `ci-build-specialist` | `agents/ci-build` | [#3](https://github.com/rakmahefa/dust/pull/3) | Comment sent; awaiting agent activity |
| W4 — Native profiling | `performance-engineering-specialist` | `agents/native-performance` | [#4](https://github.com/rakmahefa/dust/pull/4) | Comment sent; awaiting agent activity |
| W5 — WebAssembly frontend | `wasm-frontend-specialist` | `agents/wasm-frontend` | [#5](https://github.com/rakmahefa/dust/pull/5) | Comment sent; awaiting agent activity |

Each PR contains a scoped `.agent-tasks/*.md` handoff. Keep all five PRs in draft until a human has reviewed implementation, tests and reported limitations. Do not merge agent output automatically.

## Phase 0 — WGPU 3D correctness gate

### W1 — WGPU rendering specialist

- [ ] Trace GX frame preparation through `RendererTx::swap_buffers`, `start_rendering`, triple-buffer handoff and the WGPU render worker.
- [ ] Identify the reason `renderer.render_frame(...)` / queue submission is commented out in `render/wgpu-3d/src/threaded.rs`.
- [ ] Verify the current WGPU version/API and the upstream FIXME context; do not re-enable submission blindly.
- [ ] Validate frame index progression, worker wake-up/park lifecycle and command-buffer ownership.
- [ ] Confirm color output view updates and synchronization with the 2D renderer/UI.
- [ ] Reproduce with a redistributable 3D homebrew/test ROM or deterministic test scene.
- [ ] Fix the rendering path or explicitly gate it with a clear diagnostic if the implementation cannot be safely restored.
- [ ] Add regression coverage or document the exact manual validation procedure.
- [ ] Report backend, adapter, OS, test workload and validation result.

**Acceptance:** WGPU 3D frames are actually rendered/submitted and displayed without stale frames, deadlocks or ownership races; otherwise the backend fails clearly and is not presented as functional.

## Phase 1 — Correctness and tests

### W2 — Emulator QA specialist

- [ ] Inventory existing inline unit tests, test utilities and testable public/internal APIs.
- [ ] Identify high-risk deterministic invariants in CPU scheduling, memory/bus access, DMA/timers, GPU state and savestates.
- [ ] Add focused tests that do not require commercial game ROMs.
- [ ] Define a smoke-test workflow using legally redistributable homebrew/test ROMs.
- [ ] Record how to obtain, run and validate each test fixture.
- [ ] Check test determinism and isolation; avoid timing-sensitive tests without an explicit clock/control strategy.
- [ ] Document test coverage gaps that cannot reasonably be automated yet.
- [ ] Provide a concise local validation command set.

**Acceptance:** clean-checkout tests are repeatable, do not depend on proprietary ROMs, and fail with actionable context.

## Phase 2 — CI and build reproducibility

### W3 — CI/build specialist

- [ ] Remove unconditional `cargo update` from the Clippy and native build workflows.
- [ ] Use `--locked` on Cargo commands where the repository/toolchain supports it.
- [ ] Add and commit a Node package lockfile for `frontend/web`.
- [ ] Replace `npm install` with `npm ci` in CI.
- [ ] Add a formatting check using `cargo fmt --all -- --check`.
- [ ] Add a dedicated test job using the agreed workspace/package scope.
- [ ] Keep Linux, Windows and macOS build coverage and explicit feature combinations.
- [ ] Separate validation on pull requests from web deployment to the deployment branch.
- [ ] Verify artifact contents and names for each release target.
- [ ] Evaluate Linux archive versus AppImage/Flatpak packaging; document target systems, runtime dependencies and trade-offs before adding a new package.
- [ ] Record CI run links and exact failing steps after the workflow is exercised.

**Acceptance:** dependency resolution is locked; PR checks include formatting and tests; deployment cannot run as a side effect of an arbitrary pull request; release artifacts are inspectable.

## Phase 3 — Native performance baseline

### W4 — Performance engineering specialist

- [ ] Add opt-in profiling/timing instrumentation without changing normal emulator timing.
- [ ] Measure emulation-thread and renderer-thread time per frame.
- [ ] Measure software 2D scanline wait/spin time and renderer CPU usage.
- [ ] Measure desktop frame limiter behavior under active, paused and unlimited modes.
- [ ] Measure WGPU frame handoff, render submission and capture synchronization after W1 is validated.
- [ ] Compare AVX2 and general software 2D paths on the same x86_64 machine and workload.
- [ ] Profile CPU dispatch and bus/memory fallback paths before proposing a JIT or dispatch rewrite.
- [ ] Record median and p95 frame times, CPU utilization, frame pacing and audio underruns.
- [ ] Publish a reproducible before/after benchmark report.
- [ ] Recommend only changes supported by collected data.

**Acceptance:** baseline is repeatable and identifies measured hotspots; no optimization is merged without correctness checks and before/after evidence.

## Phase 4 — WebAssembly/frontend performance

### W5 — WebAssembly specialist

- [ ] Map the emulator worker, renderer worker and UI message protocol.
- [ ] Measure `run_frame` framebuffer conversion/allocation and transfer cost.
- [ ] Measure worker message latency, buffer lifetime and frame pacing in a supported browser.
- [ ] Verify SharedArrayBuffer/atomics requirements and COOP/COEP delivery headers.
- [ ] Determine whether framebuffer copying is material relative to emulation and rendering time.
- [ ] If justified, prototype reusable transfer buffers or shared-memory handoff with explicit producer/consumer ownership.
- [ ] Add back-pressure or bounded buffering; do not allow unbounded frame queues.
- [ ] Verify that buffers are not reused before the receiving worker/UI has completed consumption.
- [ ] Test pause, resume, stop, reset, tab visibility changes and repeated ROM loading.
- [ ] Document browser requirements and a useful unsupported-browser diagnostic/fallback.

**Acceptance:** measured improvement in a named metric with no frame corruption, worker deadlocks, memory growth or audio regressions.

## Cross-workstream integration

- [ ] Review each specialist PR for scope, tests, ownership assumptions and unsupported claims.
- [ ] Merge W1 before using WGPU 3D performance as a baseline.
- [ ] Merge W2 before relying on test commands in CI.
- [ ] Merge W3 after agreeing the stable test and formatting commands.
- [ ] Re-run full native and web build validation after integration.
- [ ] Re-run performance workloads on the integrated main branch.
- [ ] Update [status-audit.md](status-audit.md) and [performance-audit.md](performance-audit.md) with verified outcomes.
- [ ] Keep the upstream hardware `TODO.md` synchronized deliberately; do not mark hardware research complete based only on unit tests.

## Definition of done

- [ ] All P0/P1 workstreams have a reviewed PR and reproducible verification evidence.
- [ ] CI is deterministic and required checks pass.
- [ ] WGPU 3D status is unambiguous and supported by runtime evidence.
- [ ] Performance claims are backed by repeated measurements.
- [ ] Remaining limitations, unsupported targets and deferred optimizations are documented.
