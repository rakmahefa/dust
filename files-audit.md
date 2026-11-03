# Files audit

- Audit date: 2026-11-03
- Repository: [rakmahefa/dust](https://github.com/rakmahefa/dust)
- Branch: main
- Audited revision: 76d79f25e4ea3ce355c4ac868f5ac9a233e64b89
- Upstream: kelpsyberry/dust, main
- Scope: tracked Git tree at the audited revision; static inspection only

## Executive summary

The repository is a Rust workspace for a Nintendo DS emulator, with desktop and WebAssembly frontends and multiple software/GPU rendering backends.

The audited tree contains **293 tracked files**, approximately **6.59 MB**, including **235 Rust source files**. The workspace has eight members. The largest areas are the emulation core, frontends, and renderer implementations.

The fork and upstream main branches point to the same commit at audit time. The latest visible commit is dated 2025-12-17. The tree contains no dedicated tests or benches directory/file names; this does not rule out inline unit tests.

## Repository inventory

| Area | Tracked files | Purpose |
| --- | ---: | --- |
| core/ | 136 | Emulation hardware model, CPUs, memory, GPU, audio, scheduling, BIOS and peripherals |
| frontend/desktop/ | 60 | Native desktop UI, emulation lifecycle, input, audio, savestates and debugger views |
| frontend/web/ | 30 | Web UI, TypeScript workers, WebAssembly bridge and web renderer |
| render/soft-2d/ | 21 | Software 2D rendering and shared scanline implementation |
| render/soft-3d/ | 4 | Software 3D renderer |
| render/wgpu-2d/ | 13 | WGPU 2D rendering |
| render/wgpu-3d/ | 15 | WGPU 3D rendering |
| .github/workflows/ | 4 | Clippy, native release builds, macOS app bundles and web deployment |

The top-level tree also includes Cargo manifests and lockfile, README, TODO, license, game database, screenshot, rustfmt configuration and Cargo target configuration.

### File types

- Rust: 235 files
- TOML: 11 files
- YAML workflows: 4 files
- TypeScript: 9 files
- Remaining files: lockfile, JSON, HTML, JavaScript, CSS/Less, WGSL shaders, images, fonts, plist and XML resources

### Workspace

The root Cargo workspace declares eight members:

1. core
2. render/soft-2d/base
3. render/soft-2d
4. render/soft-3d
5. render/wgpu-2d
6. render/wgpu-3d
7. frontend/desktop
8. frontend/web/crate

The workspace uses resolver 2. The CI profile inherits release settings, disables debug information, enables LTO and uses one codegen unit. The regular release profile retains debug information and aborts on panic.

## Important files

| File | Role and audit notes |
| --- | --- |
| [Cargo.toml](Cargo.toml) | Workspace membership and release/CI profiles |
| [Cargo.lock](Cargo.lock) | Rust dependency resolution snapshot; CI currently runs cargo update, so the committed lockfile is not necessarily the exact resolution used by CI |
| [README.md](README.md) | Product description, frontend link, release artifact matrix and credits |
| [TODO.md](TODO.md) | Hardware research backlog, missing emulator functionality and frontend additions |
| [.cargo/config.toml](.cargo/config.toml) | WebAssembly target features and shared-memory linker arguments |
| [core/Cargo.toml](core/Cargo.toml) | Core features and dependencies |
| [core/build.rs](core/build.rs) | Build-time generation of ARM7/ARM9 ARM and Thumb instruction tables |
| [core/src/emu.rs](core/src/emu.rs) | Emulator state, shared scheduling and CPU/device execution loop |
| [core/src/cpu/interpreter.rs](core/src/cpu/interpreter.rs) | Interpreter engine and ARM7/ARM9 interpreter data |
| [core/src/cpu/jit.rs](core/src/cpu/jit.rs) | Currently a TODO stub despite the feature plumbing |
| [frontend/desktop/src/emu.rs](frontend/desktop/src/emu.rs) | Native emulation thread, frame loop, messages, save flushing and pacing |
| [frontend/desktop/src/ui.rs](frontend/desktop/src/ui.rs) | Desktop app, renderer selection and emulation-thread lifecycle |
| [frontend/web/crate/src/lib.rs](frontend/web/crate/src/lib.rs) | wasm-bindgen interface and WebAssembly emulator frame entry point |
| [frontend/web/src/emu/emu.ts](frontend/web/src/emu/emu.ts) | Web emulator worker and frame/audio messaging |
| [render/soft-2d/src/threaded/lockstep_scanlines.rs](render/soft-2d/src/threaded/lockstep_scanlines.rs) | Threaded software scanline renderer |
| [render/wgpu-3d/src/threaded.rs](render/wgpu-3d/src/threaded.rs) | Threaded WGPU 3D renderer handoff |
| [.github/workflows/build-release.yml](.github/workflows/build-release.yml) | Windows, Linux and macOS binary build matrix |
| [.github/workflows/build-release-macos-app-bundles.yml](.github/workflows/build-release-macos-app-bundles.yml) | Universal macOS app bundle build |
| [.github/workflows/clippy.yml](.github/workflows/clippy.yml) | Clippy matrix on Windows, Linux and macOS |
| [.github/workflows/deploy-web.yml](.github/workflows/deploy-web.yml) | WebAssembly/frontend build and deployment |

## Architecture map

### Emulation core

The core contains ARM7 and ARM9 CPU implementations, interpreter dispatch, bus and memory handling, scheduling, DMA, timers, interrupt handling, GPU/2D/3D engines, audio, DS cartridge and SPI interfaces, RTC, flash and savestate support.

The build script uses the arm-decoder build dependency to emit instruction dispatch tables into Cargo's OUT_DIR. This is build-time code generation; it should not be confused with a working runtime JIT.

### Desktop frontend

The desktop frontend launches an emulation thread and communicates with the UI through crossbeam channels. Frame data is passed through a triple buffer. Renderer backends include synchronous software rendering, lockstep threaded software rendering and WGPU paths. Audio input/output have separate backend code.

### Web frontend

The web frontend has separate workers for emulation and 3D rendering. The WebAssembly target is configured for atomics and shared memory. The deployment headers set Cross-Origin-Opener-Policy and Cross-Origin-Embedder-Policy, which are necessary for the intended shared-memory browser environment.

### Rendering

The renderer workspace separates reusable 2D software code, software 3D, WGPU 2D and WGPU 3D. The software 2D tree includes x86_64 AVX2-specific modules as well as general implementations. The existence of those modules is confirmed; the runtime dispatch path and benefit on representative workloads should be benchmarked.

## Repository hygiene and coverage observations

- No dedicated tests/ or benches/ directory, and no test- or benchmark-named Rust source file, was found in the tracked tree.
- This inventory does not prove that no inline unit tests exist. The repository's code-search endpoint did not provide usable test matches during this audit.
- The workflows do not contain an explicit cargo test or cargo fmt --check step.
- The web package has package.json but no package-lock.json in the audited tree. The workflow uses npm install rather than npm ci.
- Workflows run cargo update. This can make CI dependency resolution differ from the committed Cargo.lock and weakens reproducibility.
- Linux release packaging currently produces a tar archive containing the native executable. No AppImage, Flatpak or distribution-package manifest/script was found in the tracked tree.
- README workflow badges and nightly artifact links point to the upstream kelpsyberry/dust repository.
- game_db.json is approximately 725 KB; screenshot.png is approximately 2.01 MB. Desktop font resources contribute additional bundled assets.

## Git state and audit limits

The fork has one branch, main, and it is not protected. Its HEAD is 76d79f25e4ea3ce355c4ac868f5ac9a233e64b89, dated 2025-12-17. Upstream kelpsyberry/dust main resolves to the same SHA, so the fork is synchronized with upstream at this exact snapshot.

The Actions runs endpoint returned total_count=0 during this audit. Therefore, this audit cannot certify a passing or failing CI run. No local build, tests, emulator execution or benchmark was performed; all findings are based on the checked-in source and workflow definitions.
