# TDD Discipline

Load when `task.protocol` is `tdd_required` or `tdd_preferred`, or when writing/changing tests and mocks.

## Core Rule

Prove the behavior with a failing reproduction before changing production code.

Prefer an automated test. When the codebase or task does not support that yet, use the smallest reproducible script or equivalent failing proof.

If mocking is required, keep the behavior under proof real.

## Fail-First Cycle

### RED - Capture One Failing Reproduction

- Reproduce one observable behavior
- Prefer a clear behavior-focused automated test when possible
- Prefer real code paths; mock only collaborators you understand

### VERIFY RED - See the Right Failure

- Run the smallest command that exercises the new reproduction
- The reproduction must fail, not crash because of setup mistakes
- The failure should match the missing behavior, not a typo or broken fixture
- If the reproduction passes immediately, you are not proving the new behavior yet

### GREEN - Implement the Smallest Change

- Change only what is needed to satisfy the failing reproduction
- Do not add extra options, refactors, or speculative cleanup in the same step
- For bug fixes, keep the failing reproduction as durable regression coverage whenever automation is possible

### VERIFY GREEN - Confirm the Fix, Not a Guess

- The new failing reproduction now passes
- Relevant impacted tests still pass
- Output stays clean enough for the chosen verification path to be trustworthy

### REFACTOR - Only After Green

- Improve names, duplication, and structure after the behavior is green
- Keep behavior unchanged during refactor
- If the refactor changes behavior, start a new RED cycle

## Protocol Expectations

### `tdd_required`

Behavior-changing implementation rounds must execute and record the failing reproduction and the command that proves it before implementation.

### `tdd_preferred`

Use the same fail-first cycle by default. If a round is pure docs, config, or other non-behavior work, record why the TDD overlay does not apply before patching.

## Anti-Patterns

### Tests After Implementation

Do not write production code first and add verification later. A reproduction that passes on first run proves much less than one that first failed for the right reason.

### Mock Placeholder Assertions

Do not assert that a mock rendered. Assert parent-observable behavior.

### Test-Only Production Seams

Do not add `_resetForTest()`, `getInternalStateForTest()`, `destroy()`, or similar test-only APIs. Put cleanup and control in test helpers.

### High-Level Mocks Before Dependency Mapping

Do not mock until you know which writes, config changes, signals, or lifecycle updates the test must still observe. Start real; mock the lowest external boundary that preserves them.

### Partial Fixtures

Fixtures and mock responses must include every field downstream code may read. Do not invent a shape that only satisfies today's assertion.

### Mocking the Unit Under Test

Do not mock the module or method whose behavior the test should establish. Mock collaborators; keep decision logic real.

### Weakening Tests to Get Green

Do not delete assertions or rewrite the test to match buggy output unless expected behavior changed.

### Manual Confirmation Without Durable Coverage

Manual reproduction is useful while investigating. It does not replace keeping a durable regression test when the codebase can support one.

## Completion Check

For `tdd_preferred` rounds that legitimately skip fail-first work, record the skip reason in `Decisions`. The checks below apply when the fail-first overlay is in scope.

Before claiming the round is complete, confirm:

1. The intended behavior was first observed as a failing reproduction.
2. The failing reproduction failed for the expected reason.
3. The implementation change stayed minimal.
4. Relevant verification now passes.
5. No production API exists only for tests.
6. Mocks and fixtures preserve the behavior under proof.

If any answer is no or unclear, fix that first.
