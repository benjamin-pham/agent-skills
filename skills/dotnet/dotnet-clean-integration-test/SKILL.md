---
name: dotnet-clean-integration-test
description: >
  Generate integration test project and test classes for ASP.NET Core Clean Architecture
  projects — xUnit + WebApplicationFactory + Testcontainers (PostgreSQL) + Respawn + FluentAssertions.
  Creates a single IntegrationTests project, sets up CustomWebApplicationFactory with a real test
  database via Docker, configures Respawn for clean state between tests, generates JWT token helpers
  for protected endpoints, and generates both API endpoint tests and repository tests.
  Trigger whenever the user wants integration tests, API endpoint tests, WebApplicationFactory setup,
  repository tests with a real database, Testcontainers setup, or full end-to-end test infrastructure
  — including Vietnamese like "tạo integration test", "viết integration test", "test endpoint API",
  "test repository với database thật", "thêm integration test project", "test toàn bộ luồng".
  Do NOT trigger for unit tests (no DB, no HTTP), which belong in the dotnet-clean-unit-test skill.
---

# ASP.NET Core — Clean Architecture Integration Test Generator

Generates a complete integration test project that tests your application end-to-end:
- **API endpoint tests** — spin up the real ASP.NET Core app, call HTTP endpoints, assert HTTP status and JSON body
- **Repository tests** — test EF Core repositories against a real PostgreSQL database in Docker

All tests use a shared Testcontainers database started once per test run, with Respawn wiping data between tests so each test starts clean.

## Scope

| Test Type | What it tests | Tool |
|-----------|--------------|------|
| **Endpoint tests** | HTTP endpoint → Application → Infrastructure → DB | `WebApplicationFactory` + real HTTP client |
| **Repository tests** | Repository implementation → EF Core → real DB | Direct repository call via DI scope |

Unit tests (handlers, domain logic, validators) belong in the `dotnet-clean-unit-test` skill.

## Workflow

### Step 1 — Detect Project Structure

Find the `.slnx` file and identify `{ProjectName}`. Verify these exist:
```
src/{ProjectName}.API/          → will reference this for WebApplicationFactory
src/{ProjectName}.Infrastructure/ → has AppDbContext and repositories
```

Check what features/entities exist so you know what test classes to generate:
- Minimal API endpoints in `API/Endpoints/` or controllers in `API/Controllers/`
- Repository interfaces in `Domain/Abstractions/`
- `AppDbContext` in `Infrastructure/Data/`

### Step 2 — Create the IntegrationTests Project

Read `references/test-project-setup.md` for the full setup. This step:
1. Creates `tests/{ProjectName}.IntegrationTests/` project
2. Adds packages to `Directory.Packages.props`
3. Adds project reference to the solution
4. Exposes `Program` class for `WebApplicationFactory`

### Step 3 — Set Up Test Infrastructure

Read `references/webappfactory-setup.md` for the complete implementation. This step creates:

- **`CustomWebApplicationFactory.cs`** — starts PostgreSQL container via Testcontainers, replaces the real `DbContext` connection string with the test container's, applies migrations
- **`IntegrationTestBase.cs`** — base class all test classes inherit; holds `HttpClient`, `Respawn` instance, `SeedAsync` helpers, and `AuthenticateAs` for JWT
- **`Helpers/JwtTokenHelper.cs`** — generates signed JWT tokens using the app's own key from config, so tests can authenticate as any user/role
- **`GlobalUsings.cs`** — global using declarations

### Step 4 — Generate Test Classes

Based on what the user asks for, two scenarios:

**Scenario A — "Set up integration tests for my project"** (broad request):
- Complete Step 2 + Step 3 first
- Scan features: find each endpoint group and each repository interface
- For each endpoint group → generate `{Feature}EndpointTests.cs`
- For each repository interface → generate `{Feature}RepositoryTests.cs`

**Scenario B — "Write integration tests for ProductsEndpoint"** (specific request):
- Ensure project and test infrastructure exist (create if needed)
- Read the target endpoint/repository to understand its routes, inputs, and outputs
- Generate exactly one test file

Read `references/endpoint-test-templates.md` for endpoint test class templates.
Read `references/repository-test-templates.md` for repository test class templates.

### Step 5 — Validate

```bash
dotnet build
dotnet test tests/{ProjectName}.IntegrationTests
```

The first run will pull the Docker image — this can take 30–60 seconds. Fix any compilation or test failures before finishing.

---

## Test Writing Guidelines

### Test Class Structure

Every test class follows the same pattern — inherit `IntegrationTestBase`, use primary constructor:

```csharp
public class ProductsEndpointTests(CustomWebApplicationFactory factory)
    : IntegrationTestBase(factory)
{
    // Tests here
}
```

### Naming Convention

`MethodOrRoute_StateUnderTest_ExpectedBehavior`:

```csharp
GetAll_WithSeededProducts_ReturnsOkWithList()
GetById_WithNonExistingId_ReturnsNotFound()
Create_WithValidCommand_ReturnsCreatedAndPersists()
Create_WithMissingName_ReturnsUnprocessableEntity()
Update_AsUnauthorizedUser_ReturnsUnauthorized()
```

### Arrange-Act-Assert

Every test uses explicit Arrange / Act / Assert blocks:

```csharp
[Fact]
public async Task Create_WithValidCommand_ReturnsCreatedAndPersists()
{
    // Arrange
    AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
    var command = new CreateProductCommand("Widget", "A great widget", 29.99m);

    // Act
    var response = await Client.PostAsJsonAsync("/api/products", command);

    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.Created);
    var id = await response.Content.ReadFromJsonAsync<Guid>();
    id.Should().NotBeEmpty();

    var saved = await GetFromDbAsync(db => db.Products.FindAsync(id).AsTask());
    saved!.Name.Should().Be("Widget");
}
```

### What to Test Per Endpoint Group

For a CRUD feature with `GET /api/products`, `GET /api/products/{id}`, `POST`, `PUT`, `DELETE`:

| Test | What it checks |
|------|---------------|
| `GetAll_WithSeededProducts_ReturnsOkWithList` | happy path: seeds data, calls GET /, verifies count and shape |
| `GetAll_WithNoData_ReturnsOkWithEmptyList` | empty state |
| `GetById_WithExistingId_ReturnsOk` | happy path: returns correct data |
| `GetById_WithNonExistingId_ReturnsNotFound` | 404 guard |
| `Create_WithValidCommand_ReturnsCreatedAndPersists` | happy path: creates, verifies DB |
| `Create_WithInvalidData_ReturnsUnprocessableEntity` | FluentValidation rejection |
| `Create_WhenUnauthenticated_ReturnsUnauthorized` | auth guard |
| `Update_WithValidCommand_ReturnsNoContent` | happy path: updates, verifies DB |
| `Update_WithNonExistingId_ReturnsNotFound` | 404 guard |
| `Delete_WithExistingId_ReturnsNoContent` | happy path: deletes, verifies DB gone |

You don't need all of these for every feature — generate what makes sense for the specific entity and authorization model.

### Seeding Data

Use `SeedAsync` from `IntegrationTestBase` to insert test data before each test:

```csharp

var product = Product.Create("Test", 9.99m, categoryId);
await SeedAsync(product);

// Multiple entities
var products = new[]
{
    Product.Create("A", 1m, categoryId),
    Product.Create("B", 2m, categoryId),
};
await SeedAsync(products);
```

`SeedAsync` adds the entity to the DbContext and calls SaveChangesAsync — the entity's `Id` (set inside `Create()`) is available immediately after the call, so you can use `product.Id` in subsequent assertions.

### Authentication

For protected endpoints, call `AuthenticateAs` before making the request. It attaches a valid JWT to `Client.DefaultRequestHeaders`:

```csharp
AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
var response = await Client.PostAsJsonAsync("/api/products", command);
```

For unauthenticated tests, do NOT call `AuthenticateAs` — `Client` starts without credentials.

### Reading Response Bodies

Use `System.Net.Http.Json` extensions already available via `GlobalUsings`:

```csharp
// Read typed body
var result = await response.Content.ReadFromJsonAsync<ProductDto>();

// Read list
var list = await response.Content.ReadFromJsonAsync<List<ProductDto>>();

// Read raw string for debugging
var raw = await response.Content.ReadAsStringAsync();
```

### FluentAssertions Patterns

```csharp
// HTTP status
response.StatusCode.Should().Be(HttpStatusCode.OK);
response.StatusCode.Should().Be(HttpStatusCode.Created);
response.StatusCode.Should().Be(HttpStatusCode.NoContent);
response.StatusCode.Should().Be(HttpStatusCode.NotFound);
response.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);

// Response body
result.Should().NotBeNull();
result!.Name.Should().Be("Widget");
list.Should().HaveCount(2);
list.Should().ContainSingle(p => p.Name == "Widget");

// DB verification
var saved = await GetFromDbAsync(db => db.Products.FindAsync(id).AsTask());
saved.Should().NotBeNull();
saved!.IsDeleted.Should().BeFalse();

// Verify deleted
var deleted = await GetFromDbAsync(db => db.Products.FindAsync(id).AsTask());
deleted.Should().BeNull();
```

---

## Important Reminders

- Use **file-scoped namespaces** everywhere
- Use **C# 13** features: primary constructors, collection expressions
- `CustomWebApplicationFactory` is shared across all tests in a class via `IClassFixture` — **never modify** it inside a test
- Respawn resets data between tests, but the Docker container stays alive for the full test run — this is intentional and fast
- Always verify DB state after mutations (POST/PUT/DELETE) — don't just check the HTTP status
- If the project uses SQL Server instead of PostgreSQL, swap `Testcontainers.PostgreSql` for `Testcontainers.MsSql` in `test-project-setup.md` and update the container builder in `webappfactory-setup.md`
- If user provides entity or feature name in Vietnamese, translate to English PascalCase and confirm

## References

- `references/test-project-setup.md` — packages, .csproj, project creation commands, Program.cs exposure
- `references/webappfactory-setup.md` — CustomWebApplicationFactory, IntegrationTestBase, JwtTokenHelper complete implementations
- `references/endpoint-test-templates.md` — full endpoint test class template with all CRUD cases
- `references/repository-test-templates.md` — repository test class template
