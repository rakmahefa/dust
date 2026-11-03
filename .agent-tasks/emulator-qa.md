# Agent handoff

**Specialist:** `emulator-qa-specialist`  
**Target branch:** `agents/emulator-qa`  
**Base:** `main`  
**Task:** P1 — Establish deterministic emulator correctness tests

# Assigned work: emulator QA and test foundation

Use repository custom agent **`emulator-qa-specialist`**.

## Context
The static audit found no dedicated `tests/` or `benches/` tree. Inline tests may still exist, so inventory before concluding that test coverage is absent. The existing upstream TODO contains numerous hardware-research items.

## Required work
1. Inventory inline tests, utilities and testable core seams.
2. Select a small, high-value set of deterministic invariants around scheduler/state transitions, CPU-visible state, memory/bus, DMA/timers, savestate round trips and renderer state where practical.
3. Add focused tests with controlled inputs; do not change emulation timing or semantics to make tests pass.
4. Define a smoke-test procedure using legally redistributable homebrew/test ROMs only. Do not add commercial ROMs or proprietary BIOS files.
5. Document setup, commands, expected outcomes and gaps that still need hardware/manual validation.
6. Do not edit CI workflow files in this task; record the exact commands for the CI specialist.

## Acceptance criteria
- Tests execute from a clean checkout.
- Tests are deterministic and do not require commercial ROMs.
- The report separates automated coverage from manual/hardware validation and identifies remaining gaps.

## Repository guardrails

- Read `implementation-plan.md`, `execution-todo.md` and the relevant audits first.
- Work only within the assigned scope.
- Run the narrowest relevant tests, formatting and build checks.
- Do not merge the PR. Request review and report any blockers.
- Update this handoff with files changed, commands executed, results and known limitations.
