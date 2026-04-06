---
applyTo: '**/*'
description: 'Testing strategy and requirements'
---

# Testing guidelines

## General principles

- Every new service method, endpoint, or component MUST have corresponding tests.
- Tests MUST be deterministic — no randomness, no external service calls, no timing dependencies.
- Tests MUST be independent — no shared mutable state, no ordering assumptions.
- Follow the Arrange / Act / Assert pattern consistently.
- Prefer testing behaviour over implementation details.
- Keep tests fast — avoid unnecessary I/O, sleeping, or heavy setup.

---

## Backend testing (.NET / xUnit)

### Framework and packages

- **xUnit** for test authoring and execution.
- **Microsoft.NET.Test.Sdk** for test host integration.
- **coverlet.collector** for code coverage.
- Test project: `src/Application.Tests/`.

### Naming conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Test class | `{ClassUnderTest}Tests` | `PrimitiveServiceTests` |
| Test method | `{Method}_{Scenario}_{ExpectedBehavior}` | `CreatePrimitiveAsync_DuplicateName_ThrowsDomainException` |
| Test file | Match class name | `PrimitiveServiceTests.cs` |

### Test structure

- Use `[Fact]` for single-case tests.
- Use `[Theory]` with `[InlineData]` or `[MemberData]` for parameterised tests.
- Mark test classes `sealed`.
- One test class per file, one concept per test method.

### Isolation strategy

- Use **in-memory repository implementations** (sealed private classes inside the test class) to isolate application services from infrastructure.
- Use `NullLogger<T>.Instance` for logger dependencies.
- Pass `CancellationToken.None` for async operations.
- Do NOT use mocking frameworks (e.g., Moq) when an in-memory implementation is practical.
- Do NOT depend on a running database or external service.

### What to test

- Application service methods (business logic, mapping, validation).
- Domain entity behaviour and value-object invariants.
- Edge cases: null inputs, empty collections, boundary values, duplicate detection.
- Error paths: expected domain exceptions, validation failures.

### What NOT to test in unit tests

- EF Core configurations and migrations — validated through integration/HTTP tests.
- FastEndpoints routing wiring — validated through HTTP tests.
- Third-party library internals.

---

## API integration tests (HTTP files)

### Location and purpose

- HTTP test files live in `src/API/tests/http/`.
- One `.http` file per resource / feature area.
- HTTP files serve as **living API documentation** and **manual integration tests**.

### Conventions

- Define `@API_HostAddress` variable at the top of each file.
- Include `Authorization: Bearer {{token}}` header with a placeholder.
- Cover: success path, validation errors, not-found, and duplicate/conflict scenarios.
- Add comments explaining each request's purpose.
- Use variable substitution (`@id = …`) for chaining dependent requests.
- Keep the checked-in OpenAPI schema at `docs/openapi/studio-api-v1.json` in sync with API contract changes by running `pwsh ./scripts/export-openapi.ps1`.

### When to create or update HTTP files

- When creating or modifying an API endpoint, always create or update the corresponding `.http` file.
- New resource areas get a new `.http` file; existing areas get updated in-place.
- When request/response DTOs or route contracts change, update the relevant `.http` files and regenerate `docs/openapi/studio-api-v1.json`.

---

## Test hygiene

- Clean up global stubs and mocks in `afterEach` — never leak state between tests.
- Commit all test artifacts alongside the code they test.
- Do not skip or disable tests without a tracked issue.
- Failing tests block merges — fix the test or the code, never delete the test.
