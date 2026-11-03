# Agent handoff

**Specialist:** `ci-build-specialist`  
**Target branch:** `agents/ci-build`  
**Base:** `main`  
**Task:** P1 — Make Rust and WebAssembly CI reproducible

# Assigned work: CI and build reproducibility

Use repository custom agent **`ci-build-specialist`**.

## Context
The audit found that CI invokes `cargo update`, the web workflow uses `npm install` and the tracked tree has no Node lockfile. The workflows do not explicitly run `cargo fmt --check` or `cargo test`. Linux release output is a tar archive, not an AppImage.

## Required work
1. Remove routine `cargo update` from build/lint workflows and use locked resolution (`--locked`) where supported.
2. Add the correct package-manager lockfile for `frontend/web` and use `npm ci`.
3. Add formatting and test validation. Coordinate test command scope with the QA handoff in the sibling PR; do not assume a command that has not been validated.
4. Preserve Windows/Linux/macOS native matrix and Wasm build requirements.
5. Make deployment occur only for the intended branch/event; PR validation must not publish deployment artifacts.
6. Check artifact paths, names and archive contents.
7. Evaluate Linux package options (existing tar archive vs AppImage/Flatpak) and document requirements/trade-offs. Do not add a package blindly.
8. Avoid unrelated toolchain upgrades.

## Acceptance criteria
- Dependency resolution is reproducible.
- PR CI includes formatting and validated tests.
- Web installation is lockfile-based.
- Deployment and artifact conditions are explicit.
- Report actual CI run links/results or state clearly when execution was unavailable.

## Repository guardrails

- Read `implementation-plan.md`, `execution-todo.md` and the relevant audits first.
- Work only within the assigned scope.
- Run the narrowest relevant tests, formatting and build checks.
- Do not merge the PR. Request review and report any blockers.
- Update this handoff with files changed, commands executed, results and known limitations.
