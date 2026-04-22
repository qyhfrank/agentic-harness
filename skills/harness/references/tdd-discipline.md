# TDD Discipline

Load when writing/changing tests or adding mocks.

## Core Rule

Tests prove observable behavior. If mocking is required, keep the behavior under proof real.

## Anti-Patterns

### Mock Placeholder Assertions

Do not assert a mock rendered. Assert parent-observable behavior.

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

## Pre-Dispatch Check

Before writing tests or adding mocks, confirm:

1. Assertion proves system behavior, not mock presence.
2. No production API exists only for tests.
3. Required side effects are mapped before choosing a mock boundary.
4. Unit under test stays real; only collaborators are mocked.
5. Fixtures match real dependency shape.
6. Failing tests still encode intended behavior.

If any answer is no or unclear, fix that first.
