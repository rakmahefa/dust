# Performance audit

- Audit date: 2026-11-03
- Repository: [rakmahefa/dust](https://github.com/rakmahefa/dust)
- Audited revision: 76d79f25e4ea3ce355c4ac868f5ac9a233e64b89
- Method: static inspection of the CPU execution loop, desktop/web frame paths, renderer threading and build profiles

## Executive summary

The architecture already separates several workloads: desktop emulation runs on its own thread, software 2D rendering has a scanline worker, WGPU 3D has a renderer worker, audio uses separate backend handling, and the web frontend uses workers.

The most important performance work is not to add threads indiscriminately. The ARM7 and ARM9 cores share emulated time, buses and devices and are deliberately scheduled in sequence. Renderer synchronization, busy-wait behavior, frame copies and the web frame transfer are better candidates for measurement.

**No FPS, frame-time, CPU, GPU, memory or audio-underrun benchmark was executed for this audit.** All performance observations below are code-level hypotheses or confirmed implementation facts, not measured bottlenecks.

## Findings

| Priority | Area | Finding | Recommended action |
| --- | --- | --- | --- |
| P0 | WGPU 3D correctness / throughput | The active threaded WGPU 3D worker has the call to renderer.render_frame commented out behind a FIXME. A render_frame implementation exists elsewhere in the crate, but this worker does not call it at the inspected submission point. This may be a functional blocker rather than a mere slowdown. | Reproduce with a known 3D test ROM and WGPU backend. Fix or explicitly disable/gate the path before comparing its performance. |
| P1 | Software 2D synchronization | The lockstep scanline worker spins with hint::spin_loop while processing_line is false, and the emulation side also spins while waiting for a scanline. This reduces wake-up latency but can consume CPU while waiting for the next scanline. | Measure total CPU use, emulation-thread time and renderer-thread time. Compare the current strategy with a blocking or adaptive wait while checking frame pacing and correctness. |
| P1 | Desktop frame pacing | The desktop frame loop sleeps until approximately 1 ms before its deadline, then busy-spins until the target instant. This is a precision/latency trade-off that may increase idle CPU use. | Measure CPU use while paused and while frame-limited. Evaluate a shorter spin window or platform-aware wait only if measurements show material power cost. |
| P1 | Web frame handoff | The WebAssembly run_frame method creates a Uint32Array from the framebuffer slice for every frame. That allocates and copies a full two-screen image before the typed-array buffer is transferred to the UI worker. | Profile allocation and copy time. Consider reusable shared-memory frame buffers or a bounded buffer pool, preserving ownership and avoiding a read/write race with the renderer. |
| P2 | Core CPU execution | The core run macro executes ARM9, then ARM7, then drains scheduled device events. The two emulated CPUs are not parallelized. | Keep the shared scheduler deterministic. Profile instruction dispatch, bus accesses, memory fallback handlers and device scheduling before attempting structural changes. |
| P2 | Runtime JIT | The interpreter uses generated ARM/Thumb dispatch tables. The optional jit feature also has build-time table generation, but core/src/cpu/jit.rs is only TODO. There is no completed runtime JIT visible in this snapshot. | Do not treat JIT as an available optimization. First establish CPU profiles and expand correctness tests; a JIT would be a separate, high-risk project. |
| P2 | Software renderer SIMD | x86_64 AVX2 implementations exist alongside general 2D rendering implementations. Static inspection did not establish which path is selected for each host or the measured speedup. | Verify feature detection/dispatch and benchmark scalar versus AVX2 on identical scenes and hardware. |
| P2 | Desktop framebuffer transfer | For the software 2D path, the desktop emulation loop copies the renderer framebuffer into the frame triple buffer each frame. This is a predictable memory-copy cost. | Measure its share of frame time before considering a shared framebuffer or a different ownership model. |
| P2 | Build reproducibility | CI runs cargo update before builds and Clippy; web uses npm install without a tracked package lock. This can cause changing dependency resolution and less stable build-time comparisons. | Pin CI to Cargo.lock and add a JS lockfile with npm ci before collecting build-time baselines. |

## Threading and synchronization map

### Native desktop

- UI thread: owns the desktop UI and renderer configuration.
- Emulation thread: created by frontend/desktop/src/ui.rs and runs frontend/desktop/src/emu.rs.
- Software 2D renderer: optional lockstep scanline worker.
- WGPU 3D renderer: separate worker using a triple buffer and park/unpark handoff.
- Audio: separate CPAL-side handling with a ring-buffer-style sample handoff.

The CPU cores themselves run sequentially in the core scheduler. Moving ARM7 and ARM9 to independent host threads is not a recommended first optimization because both cores advance against shared emulated time and shared devices.

### Web

- The emulator is run in a Web Worker.
- A separate renderer worker is started for 3D rendering.
- The build enables WebAssembly atomics/shared memory, and the web deployment headers opt into cross-origin isolation.

The per-frame framebuffer conversion in the WebAssembly bridge is a concrete copy/allocation point to profile. Any zero-copy approach must explicitly define which worker owns each buffer and when the UI may read it.

## Measurement plan

Use the same ROMs, save states, settings, machine and build revision for each comparison. Use homebrew or otherwise authorized test ROMs; do not commit copyrighted game ROMs.

### Workload matrix

1. CPU-heavy 2D scene, software renderer.
2. 2D scene with scrolling backgrounds, objects, blending and capture.
3. 3D-heavy scene with software 3D.
4. 3D-heavy scene with WGPU 3D, only after confirming the submission path works.
5. Audio-active scene with audio synchronization enabled and disabled.
6. WebAssembly run with the emulator worker and renderer worker enabled.
7. Paused desktop session to assess frame-limit idle cost.
8. Cold launch and game-load measurements, separately from steady-state emulation.

### Metrics

Collect at least:

- Median and p95 frame time, not only average FPS
- Emulation CPU time per frame
- 2D and 3D renderer CPU time per frame
- Host process CPU utilization and per-thread utilization
- GPU utilization and GPU frame/submission time where available
- Frame pacing variance and missed frame deadlines
- Audio underruns, buffer occupancy and synchronization waits
- Web worker message time, framebuffer copy time and allocation volume
- Cold build time and incremental build time for the native and Wasm targets

Warm up the workload first, then record multiple fixed-duration runs. Keep compiler profile, features, target triple and graphics backend identical. Report medians and spread; do not compare numbers from different hardware as if they were controlled measurements.

## Recommended implementation order

### Phase 1 — Establish correctness and baseline

- Reproduce the WGPU 3D rendering path with a known test workload.
- Verify whether the commented frame submission is an intentional temporary disablement.
- Add lightweight timing instrumentation behind an opt-in feature or diagnostic mode.
- Record baseline frame-time and CPU/GPU metrics for software and GPU paths.
- Add repeatable smoke tests so that performance changes can be checked against correctness.

### Phase 2 — Reduce demonstrated synchronization costs

- Quantify scanline busy-wait CPU cost and compare adaptive/blocking waits.
- Quantify the desktop final spin window under active and paused frame limiting.
- Profile WGPU capture synchronization separately from ordinary frame rendering.
- Do not replace atomics, unsafe shared buffers or triple-buffer ownership without a race/correctness review.

### Phase 3 — Reduce transfer and allocation overhead

- Measure the WebAssembly framebuffer copy and worker messaging.
- Prototype reusable frame buffers or shared-memory exchange only if the copy is material.
- Measure desktop software framebuffer copying before changing the triple-buffer data model.

### Phase 4 — Optimize CPU and renderer hot paths

- Profile ARM instruction dispatch, memory access/fallback code and device scheduling.
- Benchmark the x86_64 AVX2 rendering implementations against the general path.
- Evaluate LTO and codegen-unit settings against build time and runtime performance rather than assuming the most aggressive build profile is always best.
- Consider a runtime JIT only after test coverage, CPU profiles and a separate design for invalidation, timing and debugger compatibility exist.

## Build-time considerations

The CI profile inherits release, disables debug information, enables LTO and fixes codegen-units to 1. These settings can improve generated-code optimization but can increase clean build time and constrain parallel code generation. The current CI scripts also call cargo update, making repeated build-time comparisons less reproducible.

The web build uses webpack production optimization and a nightly Rust toolchain. Build-time investigations should separate Rust compilation, wasm-pack, TypeScript/webpack and artifact deployment stages.

## Acceptance criteria for future optimization work

An optimization should not be accepted solely because a local FPS counter increased. Require:

- No regression in core correctness or rendering output for the tested cases
- Repeated measurements on the same machine and revision
- Improvement in a named metric (for example p95 frame time or CPU time per emulated frame)
- No unacceptable increase in audio underruns, frame pacing variance or memory use
- Clear fallback behavior for unsupported CPUs, GPUs and browsers
- A before/after record including backend, features, compiler version and workload

## Limits

This audit cannot identify the current top runtime bottleneck without profiling. Static indicators such as spin loops, copies and thread boundaries are candidates for measurement, not proof of a user-visible slowdown. The WGPU 3D commented submission call is a source-level fact and should be investigated as a correctness issue before drawing performance conclusions.
