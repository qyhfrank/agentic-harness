# Testing Anti-Patterns

Load this reference when a task involves writing or changing tests, adding mocks, or considering test-only seams.

## Core Rule

Tests must verify real behavior, not mock behavior.

## Anti-Pattern 1: Testing Mock Existence

Bad:

```typescript
test("renders sidebar", () => {
  render(<Page />);
  expect(screen.getByTestId("sidebar-mock")).toBeInTheDocument();
});
```

Why it fails:

- It proves the mock exists, not that the system behavior is correct.
- It hides whether the real component contract still works.

Preferred:

- Test the real component when practical.
- If isolation requires a mock, assert the parent system's behavior, not the mock's placeholder output.

## Anti-Pattern 2: Test-Only Methods in Production Code

Bad:

```typescript
class Session {
  async destroy() {
    await this.workspaceManager.destroyWorkspace(this.id);
  }
}
```

Why it fails:

- It pollutes production APIs with test-only lifecycle controls.
- It confuses ownership and creates risky surface area.

Preferred:

- Put cleanup helpers in test utilities.
- Keep production types focused on production responsibilities.

## Anti-Pattern 3: Mocking Without Understanding Side Effects

Bad:

- Mocking a high-level helper before understanding which file writes, config updates, or lifecycle signals the test depends on.

Why it fails:

- The mock can silently remove the behavior the test needs to observe.
- The test then passes or fails for the wrong reason.

Preferred:

- Run with the real implementation first when practical.
- Identify the actual slow or external dependency.
- Mock at the lowest level that preserves the behavior under test.

## Anti-Pattern 4: Incomplete Mocks

Bad:

- Creating a mock response with only the fields the immediate test uses.

Why it fails:

- Downstream code may rely on omitted fields.
- The test stops reflecting the real shape of the dependency.

Preferred:

- Mirror the real schema completely.
- Include all fields that downstream consumers may inspect.

## Pre-Dispatch Checklist

Before handing a task to an implementer or writing tests yourself, ask:

1. Are we testing real behavior or just a mock placeholder?
2. Are we about to add a test-only method to production code?
3. Do we understand which side effects the test depends on?
4. Does the mock or fixture reflect the real schema closely enough?

If any answer is "no" or "not sure", fix that before proceeding.
