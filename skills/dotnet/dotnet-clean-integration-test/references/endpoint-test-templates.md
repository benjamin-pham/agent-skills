# Endpoint Test Templates

## Full CRUD Endpoint Test Class

This template covers a typical CRUD feature. Adapt it to the actual entity properties, route paths, request/response DTO shapes, and authorization requirements found in the project.

```csharp
using {ProjectName}.Application.Features.Products.Commands;

namespace {ProjectName}.IntegrationTests.Features.Products;

public class ProductsEndpointTests(CustomWebApplicationFactory factory)
    : IntegrationTestBase(factory)
{
    // ── GET /api/products ─────────────────────────────────────────────────────

    [Fact]
    public async Task GetAll_WithSeededProducts_ReturnsOkWithList()
    {
        // Arrange
        await SeedAsync([
            new Product { Name = "Widget", Price = 9.99m },
            new Product { Name = "Gadget", Price = 19.99m },
        ]);

        // Act
        var response = await Client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var list = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        list.Should().HaveCount(2);
        list.Should().ContainSingle(p => p.Name == "Widget");
    }

    [Fact]
    public async Task GetAll_WithNoData_ReturnsOkWithEmptyList()
    {
        // Act
        var response = await Client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var list = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        list.Should().BeEmpty();
    }

    // ── GET /api/products/{id} ────────────────────────────────────────────────

    [Fact]
    public async Task GetById_WithExistingId_ReturnsOkWithProduct()
    {
        // Arrange
        var product = new Product { Name = "Widget", Price = 9.99m };
        await SeedAsync(product);

        // Act
        var response = await Client.GetAsync($"/api/products/{product.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var result = await response.Content.ReadFromJsonAsync<ProductDto>();
        result!.Name.Should().Be("Widget");
        result.Price.Should().Be(9.99m);
    }

    [Fact]
    public async Task GetById_WithNonExistingId_ReturnsNotFound()
    {
        // Act
        var response = await Client.GetAsync($"/api/products/{Guid.NewGuid()}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    // ── POST /api/products ────────────────────────────────────────────────────

    [Fact]
    public async Task Create_WithValidCommand_ReturnsCreatedAndPersistsToDb()
    {
        // Arrange
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
        var command = new CreateProductCommand("Widget", "A great widget", 29.99m);

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert — HTTP
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var id = await response.Content.ReadFromJsonAsync<Guid>();
        id.Should().NotBeEmpty();

        // Assert — DB
        var saved = await GetFromDbAsync(db => db.Products.FindAsync(id).AsTask());
        saved.Should().NotBeNull();
        saved!.Name.Should().Be("Widget");
        saved.Price.Should().Be(29.99m);
    }

    [Fact]
    public async Task Create_WithInvalidData_ReturnsUnprocessableEntity()
    {
        // Arrange — Name is empty, violates FluentValidation
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
        var command = new CreateProductCommand("", "desc", 9.99m);

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
    }

    [Fact]
    public async Task Create_WhenUnauthenticated_ReturnsUnauthorized()
    {
        // Arrange — no AuthenticateAs call
        var command = new CreateProductCommand("Widget", "desc", 9.99m);

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    // ── PUT /api/products/{id} ────────────────────────────────────────────────

    [Fact]
    public async Task Update_WithValidCommand_ReturnsNoContentAndUpdatesDb()
    {
        // Arrange
        var product = new Product { Name = "Old Name", Price = 9.99m };
        await SeedAsync(product);
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
        var command = new UpdateProductCommand(product.Id, "New Name", "Updated desc", 49.99m);

        // Act
        var response = await Client.PutAsJsonAsync($"/api/products/{product.Id}", command);

        // Assert — HTTP
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Assert — DB
        var updated = await GetFromDbAsync(async db =>
        {
            db.ChangeTracker.Clear(); // bypass EF cache
            return await db.Products.FindAsync(product.Id);
        });
        updated!.Name.Should().Be("New Name");
        updated.Price.Should().Be(49.99m);
    }

    [Fact]
    public async Task Update_WithNonExistingId_ReturnsNotFound()
    {
        // Arrange
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");
        var command = new UpdateProductCommand(Guid.NewGuid(), "Name", "desc", 9.99m);

        // Act
        var response = await Client.PutAsJsonAsync($"/api/products/{command.Id}", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    // ── DELETE /api/products/{id} ─────────────────────────────────────────────

    [Fact]
    public async Task Delete_WithExistingId_ReturnsNoContentAndRemovesFromDb()
    {
        // Arrange
        var product = new Product { Name = "ToDelete", Price = 9.99m };
        await SeedAsync(product);
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");

        // Act
        var response = await Client.DeleteAsync($"/api/products/{product.Id}");

        // Assert — HTTP
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Assert — DB (should be gone or soft-deleted)
        var deleted = await GetFromDbAsync(db => db.Products.FindAsync(product.Id).AsTask());
        deleted.Should().BeNull(); // for hard delete
        // OR for soft delete: deleted!.IsDeleted.Should().BeTrue();
    }

    [Fact]
    public async Task Delete_WithNonExistingId_ReturnsNotFound()
    {
        // Arrange
        AuthenticateAs(Guid.NewGuid().ToString(), "Admin");

        // Act
        var response = await Client.DeleteAsync($"/api/products/{Guid.NewGuid()}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Notes on Adapting This Template

**Routes:** Check the actual endpoint routes in `API/Endpoints/` or `API/Controllers/`. Replace `/api/products` with the real route.

**DTOs:** Replace `ProductDto`, `CreateProductCommand`, `UpdateProductCommand` with the actual types from the `Application` layer. Read the endpoint handler to understand request/response shapes.

**Authorization:** If the endpoint uses `[Authorize(Roles = "X")]` or `.RequireAuthorization("X")`, pass the matching role in `AuthenticateAs(userId, "X")`. If the endpoint is public, skip `AuthenticateAs`.

**Validation errors:** If the project uses `FluentValidation` with problem details, the validation rejection returns `422 Unprocessable Entity`. If it uses `ModelState` only, it returns `400 Bad Request`. Check the existing `GlobalExceptionHandler` or middleware to confirm.

**Soft delete:** If the entity has `IsDeleted` flag, the delete test should verify `IsDeleted == true` rather than `null`. Check the entity definition.

**Nested resources:** For routes like `/api/categories/{categoryId}/products`, seed the parent entity first and use its `Id` in the URL.
