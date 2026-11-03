# Status audit

- Audit date: 2026-11-03
- Repository: [rakmahefa/dust](https://github.com/rakmahefa/dust)
- Audited branch/revision: main / 76d79f25e4ea3ce355c4ac868f5ac9a233e64b89
- Upstream reference: kelpsyberry/dust main

## Overall assessment

The repository contains a substantial Nintendo DS emulator implementation with native and WebAssembly frontends, multiple rendering backends and release workflows. The source tree is structured and the fork is synchronized with upstream.

The main status concern is **verification and completion**, not a confirmed global build failure: the visible revision is from 2025-12-17, no workflow run was exposed by the Actions listing at audit time, no explicit test stage exists in the workflows, the JIT implementation is a TODO stub, and the threaded WGPU 3D submission call is commented out.

No local build or runtime test was executed. Status labels below reflect static source inspection only.

## Component status

| Component | Status | Evidence / notes |
| --- | --- | --- |
| Workspace and dependency layout | Present | Eight Cargo workspace members; Cargo.lock is tracked |
| ARM7 / ARM9 emulation | Implemented baseline | CPU modules and shared scheduler exist; execution is sequential in the core run loop |
| CPU interpreter | Implemented | ARM and Thumb instruction tables are generated at build time |
| Runtime JIT | Not implemented in this snapshot | core/src/cpu/jit.rs contains only TODO; a jit feature and build-time JIT table generator exist, but they do not make the runtime JIT complete |
| 2D software rendering | Implemented, threaded option available | Synchronous and lockstep scanline renderers exist |
| 3D software rendering | Implemented with known gaps | TODO.md still lists shadow polygons and antialiasing as missing |
| WGPU 2D rendering | Implemented path present | WGPU renderer and threaded lockstep scanline backend exist |
| WGPU 3D rendering | Requires immediate verification | render/wgpu-3d/src/threaded.rs has the renderer.render_frame call commented out behind a FIXME; do not assume the accelerated 3D output is currently submitted |
| Desktop frontend | Implemented | Native UI, dedicated emulation thread, input, audio, savestates and debugger views |
| Web frontend | Implemented path present | TypeScript UI, emulation worker, renderer worker and wasm-bindgen interface |
| Audio | Implemented baseline | Core audio and desktop CPAL backends exist; TODO.md lists additional hardware behavior research |
| CI workflows | Configured, health unverified | Clippy, native builds, macOS bundles and web deployment are defined; no run history was returned in this audit |
| Test/benchmark coverage | Insufficiently evidenced | No dedicated tests/ or benches paths found; workflows omit explicit cargo test and benchmark steps |

## Build and delivery status

### Native desktop

The release workflow builds:

- Windows x86_64
- Linux x86_64 GNU
- macOS x86_64
- macOS ARM64

It builds base, debugging and debugging-plus-GDB feature configurations. A separate workflow assembles a universal macOS application bundle.

The Linux artifact is a compressed tar archive containing the desktop executable. AppImage packaging is not present in the audited tree.

### WebAssembly

The web workflow uses a nightly Rust toolchain with rust-src, Node.js 17, wasm-pack through the webpack plugin, and a production webpack build. It deploys the dist directory from main to the web-deploy branch.

The browser build expects shared-memory support: .cargo/config.toml enables Wasm atomics and shared-memory linking, and the web headers configure COOP/COEP. Deployment and browser compatibility should be tested in an actual supported browser, not inferred from compilation alone.

### CI reliability

All four workflows include workflow_dispatch. The Clippy workflow runs on Windows, Ubuntu and macOS. Build workflows compile multiple target/feature combinations.

However:

- Workflows call cargo update instead of relying strictly on the checked-in Cargo.lock.
- The web workflow uses npm install, and no package-lock.json is tracked.
- No explicit cargo test, cargo fmt --check, smoke-test ROM or performance regression stage is defined.
- The Actions runs query returned zero visible runs during this audit; current CI health is therefore unknown.

## Known incomplete functionality

TODO.md identifies ongoing hardware research across CPU timing and edge cases, DS slot/ROM, flash, graphics, RTC, SPI, audio, timers and DMA.

The documented missing functionality includes:

- Finished 3D rendering: shadow polygons and antialiasing
- Wi-Fi
- RTC alarms
- ARM7 regular open bus
- Sleep mode
- GBA slot and GBA mode
- Absent SIO behavior

The TODO also lists desired desktop improvements such as general UI polish and game controller support, plus a broader web UI rework and GPU-accelerated rendering goals.

These are project backlog entries, not independently reproduced failures.

## Repository and upstream state

- Fork branch: main only
- Branch protection: disabled
- Fork HEAD: 76d79f25e4ea3ce355c4ac868f5ac9a233e64b89
- Upstream main HEAD: the same SHA
- Commit date: 2025-12-17

The fork is therefore exactly aligned with the upstream branch at the inspected revision. The source itself is old relative to the audit date, but there is no fork-specific divergence to reconcile.

## Recommended next steps

1. **Verify the WGPU 3D path first.** Establish whether the commented frame submission is intentional, reproduce the visible behavior, and either restore a valid submission path or clearly gate the incomplete backend.
2. **Establish a test baseline.** Add deterministic core tests and a small, legally redistributable homebrew ROM smoke-test procedure. Make CI run cargo test and cargo fmt --check.
3. **Make dependency resolution repeatable.** Remove cargo update from CI; use locked Cargo builds and npm ci after adding a package lock.
4. **Record build health.** Trigger each workflow manually and retain the run URLs/artifacts so that build status is evidence-based.
5. **Document feature support.** Separate implemented, partially implemented and missing hardware behavior in project documentation.
6. **Only then plan larger optimizations.** Use the performance audit's benchmark plan before attempting JIT work or synchronization changes.

## Limitations

This document is a static audit of the checked-in tree at the specified SHA. It does not assert that every TODO is a bug, that the project cannot build, or that a particular game is playable. Build, test, CI and runtime claims require execution evidence.
