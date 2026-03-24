---
name: dotnet-clean-unit-test
description: >
  Generate unit test projects and test classes for ASP.NET Core Clean Architecture
  projects — xUnit + NSubstitute + FluentAssertions. Creates separate test projects
  per layer (Domain.Tests, Application.Tests), scaffolds test infrastructure
  (base classes, fixtures, helpers), and generates test classes with realistic
  test cases following Arrange-Act-Assert pattern.
  Trigger whenever the user wants to add unit tests, create test projects,
  write tests for entities/services/handlers, or set up a testing structure
  in a .NET clean architecture project — including Vietnamese like
  "tạo unit test", "viết test", "thêm test cho entity", "tạo test project".
  Do NOT trigger for integration tests, API tests, or WebApplicationFactory-based tests.
metadata:
  related-skills:
    - dotnet-clean-architect
---

# ASP.NET Core — Clean Architecture Unit Test Generator

Generates unit test projects and test classes for Clean Architecture .NET projects using xUnit, NSubstitute, and FluentAssertions.

## Scope

This skill covers **unit tests only** — tests that run fast, in isolation, with no external dependencies:

| Layer | What to test | Mocking |
|-------|-------------|---------|
| **Domain** | Entity behavior, domain logic, value objects, domain exceptions | No mocking needed — pure logic |
| **Application** | MediatR command/query handlers, FluentValidation validators, business rules | Mock repositories and external dependencies with NSubstitute |

Infrastructure tests (EF Core, database) and API tests (WebApplicationFactory, HTTP pipeline) are **integration tests** and belong in a separate skill.

## Workflow

### Step 1 — Detect Project Structure

Find the `.slnx` file and identify `{ProjectName}`. Scan existing layers:

```
src/{ProjectName}.Domain/          → generates tests/{ProjectName}.Domain.Tests/
src/{ProjectName}.Application/     → generates tests/{ProjectName}.Application.Tests/
```

Check what already exists:
- Entities in `Domain/Entities/`
- Enums in `Domain/Enums/`
- MediatR handlers using **vertical slice** structure: `Application/Features/{EntityPlural}/{OperationName}/`
  - e.g., `Application/Features/Products/CreateProduct/CreateProductCommandHandler.cs`
  - e.g., `Application/Features/Orders/GetOrderById/GetOrderByIdQueryHandler.cs`
- FluentValidation validators in the same operation folder as their handler

### Step 2 — Create Test Projects

Read `references/test-project-setup.md` for the complete setup guide. This step:

1. Creates test projects under `tests/` folder
2. Adds NuGet packages (xUnit, NSubstitute, FluentAssertions) to `Directory.Packages.props`
3. Adds projects to the solution with correct references
4. Creates shared test infrastructure (base classes, helpers)

Only create test projects for layers that exist in the source. If the user only has a Domain layer, only create `Domain.Tests`.

### Step 3 — Generate Test Classes

**One handler = one test file.** This is a hard rule. Never put tests for multiple handlers in the same file.

```
CreateProductCommandHandler.cs   →  CreateProductCommandHandlerTests.cs
UpdateProductCommandHandler.cs   →  UpdateProductCommandHandlerTests.cs
GetProductByIdQueryHandler.cs    →  GetProductByIdQueryHandlerTests.cs
GetAllProductsQueryHandler.cs    →  GetAllProductsQueryHandlerTests.cs
```

The test file mirrors the vertical slice path of its handler, under the test project:

```
src/MyShop.Application/Features/Products/CreateProduct/CreateProductCommandHandler.cs
tests/MyShop.Application.Tests/Features/Products/CreateProduct/CreateProductCommandHandlerTests.cs
```

Based on what the user asks for, generate test classes. There are two scenarios:

**Scenario A — "Set up unit tests for my project"** (broad request):
- Create test projects (Step 2)
- Scan existing entities, MediatR handlers, and validators
- For each handler found → generate one dedicated test file
- For each entity with domain logic → generate one test file
- For each validator → generate one test file

**Scenario B — "Write tests for CreateProductCommand handler"** (specific request):
- Ensure the test project exists (create if needed)
- Read the target handler/entity/validator to understand its behavior
- Generate exactly one test file for that handler

### Step 4 — Validate

1. Run `dotnet build` on the test projects to verify compilation
2. Run `dotnet test` to verify all tests pass
3. Fix any issues

---

## Test Writing Guidelines

These guidelines shape every test class this skill generates. The goal is tests that catch real bugs, read like documentation, and stay maintainable as the codebase evolves.

### Naming Convention

Use the pattern: `MethodName_StateUnderTest_ExpectedBehavior`

```csharp
public class ProductTests
{
    [Fact]
    public void Constructor_WithValidParameters_SetsProperties()

    [Fact]
    public void UpdatePrice_WithNegativeValue_ThrowsBadRequestException()

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void UpdatePrice_WithInvalidValue_ThrowsBadRequestException(decimal price)
}
```

This reads naturally as a specification: "UpdatePrice, with negative value, throws BadRequestException." When a test fails, the name immediately tells you what broke.

### Arrange-Act-Assert (AAA)

Every test follows this three-part structure, separated by blank lines for readability:

```csharp
[Fact]
public async Task GetByIdAsync_WithExistingId_ReturnsProduct()
{
    // Arrange
    var productId = Guid.NewGuid();
    var expected = new Product { Id = productId, Name = "Test" };
    _repository.GetByIdAsync(productId, Arg.Any<CancellationToken>())
        .Returns(expected);

    // Act
    var result = await _sut.GetByIdAsync(productId);

    // Assert
    result.Should().NotBeNull();
    result!.Name.Should().Be("Test");
}
```

### FluentAssertions Patterns

FluentAssertions makes test failures more readable — instead of "Expected True but got False", you get "Expected product.Name to be 'Widget' but found 'Gadget'."

```csharp
// Value checks
result.Should().Be(expected);
result.Should().NotBeNull();
result.Should().BeOfType<Product>();

// String checks
name.Should().StartWith("Product");
name.Should().Contain("test");
name.Should().BeNullOrWhiteSpace();

// Collection checks
products.Should().HaveCount(3);
products.Should().ContainSingle(p => p.Name == "Widget");
products.Should().BeEmpty();
products.Should().AllSatisfy(p => p.IsDeleted.Should().BeFalse());

// Exception checks
var act = () => entity.UpdatePrice(-1);
act.Should().Throw<BadRequestException>()
    .WithMessage("*negative*");

// Async exception checks
var act = async () => await service.GetByIdAsync(id);
await act.Should().ThrowAsync<NotFoundException>();
```

### NSubstitute Patterns

NSubstitute creates lightweight substitutes for interfaces — no need for complex setup. Use it in Application layer tests to isolate the class under test from its dependencies.

```csharp
// Create substitute
var repository = Substitute.For<IProductRepository>();

// Setup returns
repository.GetByIdAsync(productId, Arg.Any<CancellationToken>())
    .Returns(product);

repository.GetAllAsync(Arg.Any<CancellationToken>())
    .Returns(new List<Product> { product1, product2 });

// Returns null (for "not found" scenarios)
repository.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns((Product?)null);

// Throw async exception
unitOfWork.SaveChangesAsync(Arg.Any<CancellationToken>())
    .ThrowsAsync(new ConcurrencyException("Concurrency", new Exception()));

// Verify calls happened
await repository.Received(1).AddAsync(Arg.Is<Product>(p => p.Name == "Test"), Arg.Any<CancellationToken>());

// Verify calls did NOT happen
await repository.DidNotReceive().DeleteAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>());
```

### Handler Testing Pattern — Test Each Step Separately

The most important pattern for handler tests: **test each guard clause in isolation**. A handler has a sequence of steps — fetch A, check B, validate C, save. Write one test per failure point, and have each test build up successful context for all prior steps before testing its own failure.

```
Handler steps:          Tests:
1. Get user          → Handle_WhenUserNotFound_ReturnsFailure
2. Get apartment     → Handle_WhenApartmentNotFound_ReturnsFailure
3. Check overlap     → Handle_WhenApartmentIsAlreadyBooked_ReturnsFailure
4. Save              → Handle_WhenSaveThrows_ReturnsFailure
5. Return success    → Handle_WhenAllStepsPass_ReturnsSuccess
                     → Handle_WhenAllStepsPass_CallsRepositoryAdd (verify interaction)
```

- **Static `Command` at class level** — define the command/query once as `private static readonly` and reuse across all tests. No per-test repetition.
- **Test data factories** — create a `TestData/` folder with static `ProductData.Create()`, `UserData.Create()` methods. Much cleaner than `new Product { Name = "...", Price = 9.99m }` scattered everywhere.
- **Mock `IDateTimeProvider`** — never use `DateTime.Now` directly in tests. Always inject time as an interface and mock it: `_dateTimeProvider.UtcNow.Returns(UtcNow)`.
- **Mock `IUnitOfWork`** — if the project uses Unit of Work, test what happens when `SaveChangesAsync` throws (concurrency exception, etc.).

### What Makes a Good Unit Test

- **Test behavior, not implementation.** A test for `UpdatePrice` should verify the price changed and validation ran — not that a specific private method was called. This way, refactoring internals doesn't break tests.
- **One logical assertion per test.** Multiple `Should()` calls are fine when they verify one concept (e.g., checking all properties of a returned object). But separate tests for separate behaviors.
- **Use `[Theory]` for boundary conditions.** When testing validation with multiple invalid inputs, `[Theory]` + `[InlineData]` is cleaner than duplicating test methods.
- **Name the system under test `_sut`.** This convention makes it immediately clear which object is being tested vs. which are dependencies.
- **Don't test the framework.** EF Core, ASP.NET middleware, and xUnit itself are already tested. Focus on your business logic.

---

## Test Class Templates

Read the reference files for complete test class templates:

- `references/domain-test-templates.md` — Entity tests, domain exception tests, value object tests
- `references/application-test-templates.md` — MediatR command/query handler tests with NSubstitute (step-by-step pattern), test data factories, FluentValidation validator tests

These templates show the full file structure including usings, namespace, class setup, and test methods. Adapt them to the actual entity/service properties and methods found in the project.

## Important Reminders

- Use **file-scoped namespaces** everywhere
- Use **C# 13** features: primary constructors, collection expressions
- Always `dotnet build` test projects before running tests — catch compilation errors early
- Match the test project's `TargetFramework` to the source project (net10.0)
- Always include `CancellationToken` parameters when mocking async repository methods (use `Arg.Any<CancellationToken>()`)
- If user provides entity/service name in Vietnamese, translate to English PascalCase and confirm
