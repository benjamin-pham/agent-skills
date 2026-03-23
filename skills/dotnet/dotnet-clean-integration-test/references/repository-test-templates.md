# Repository Test Templates

Repository integration tests verify that your EF Core repository implementations work correctly against a real database. They skip the HTTP layer entirely — useful for testing custom query methods, pagination, filtering, and complex joins that are harder to assert through endpoints.

## When to Write Repository Tests

Write repository tests when the repository has **non-trivial query logic** beyond basic CRUD:
- Custom queries with filtering, sorting, or pagination
- Eager loading of navigation properties
- Queries that check `IsDeleted` for soft delete
- Bulk operations
- Methods that return DTOs or projections directly

For basic `GetByIdAsync` / `GetAllAsync` / `AddAsync` on a generic repository, the endpoint tests already exercise these indirectly. Don't duplicate that coverage.

## Repository Test Class Template

```csharp
namespace {ProjectName}.IntegrationTests.Features.Products;

public class ProductRepositoryTests(CustomWebApplicationFactory factory)
    : IntegrationTestBase(factory)
{
    // Resolve repository through the DI container within a scope
    private IProductRepository GetRepository() => GetService<IProductRepository>();

    // ── GetByIdAsync ──────────────────────────────────────────────────────────

    [Fact]
    public async Task GetByIdAsync_WithExistingId_ReturnsProduct()
    {
        // Arrange
        var product = new Product { Name = "Widget", Price = 9.99m };
        await SeedAsync(product);

        // Act
        var result = await GetRepository().GetByIdAsync(product.Id, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Name.Should().Be("Widget");
        result.Price.Should().Be(9.99m);
    }

    [Fact]
    public async Task GetByIdAsync_WithNonExistingId_ReturnsNull()
    {
        // Act
        var result = await GetRepository().GetByIdAsync(Guid.NewGuid(), CancellationToken.None);

        // Assert
        result.Should().BeNull();
    }

    // ── GetAllAsync ───────────────────────────────────────────────────────────

    [Fact]
    public async Task GetAllAsync_WithSeededData_ReturnsAll()
    {
        // Arrange
        await SeedAsync([
            new Product { Name = "A", Price = 1m },
            new Product { Name = "B", Price = 2m },
            new Product { Name = "C", Price = 3m },
        ]);

        // Act
        var result = await GetRepository().GetAllAsync(CancellationToken.None);

        // Assert
        result.Should().HaveCount(3);
    }

    [Fact]
    public async Task GetAllAsync_WithNoData_ReturnsEmptyList()
    {
        // Act
        var result = await GetRepository().GetAllAsync(CancellationToken.None);

        // Assert
        result.Should().BeEmpty();
    }

    // ── Custom query: GetByNameAsync (example) ────────────────────────────────

    [Fact]
    public async Task GetByNameAsync_WithMatchingName_ReturnsProduct()
    {
        // Arrange
        await SeedAsync(new Product { Name = "SpecificWidget", Price = 9.99m });

        // Act
        var result = await GetRepository().GetByNameAsync("SpecificWidget", CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Name.Should().Be("SpecificWidget");
    }

    [Fact]
    public async Task GetByNameAsync_WithNoMatch_ReturnsNull()
    {
        // Act
        var result = await GetRepository().GetByNameAsync("DoesNotExist", CancellationToken.None);

        // Assert
        result.Should().BeNull();
    }

    // ── Soft delete query (if the entity has IsDeleted) ───────────────────────

    [Fact]
    public async Task GetAllAsync_ExcludesSoftDeletedEntities()
    {
        // Arrange — seed one active, one soft-deleted
        await SeedAsync([
            new Product { Name = "Active", Price = 1m, IsDeleted = false },
            new Product { Name = "Deleted", Price = 2m, IsDeleted = true },
        ]);

        // Act
        var result = await GetRepository().GetAllAsync(CancellationToken.None);

        // Assert — soft-deleted entity must not appear
        result.Should().ContainSingle();
        result.Single().Name.Should().Be("Active");
    }

    // ── Navigation / eager loading (if entity has relationships) ─────────────

    [Fact]
    public async Task GetByIdAsync_IncludesCategory()
    {
        // Arrange
        var category = new Category { Name = "Electronics" };
        await SeedAsync(category);

        var product = new Product { Name = "TV", Price = 999m, CategoryId = category.Id };
        await SeedAsync(product);

        // Act
        var result = await GetRepository().GetByIdAsync(product.Id, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Category.Should().NotBeNull();
        result.Category!.Name.Should().Be("Electronics");
    }

    // ── AddAsync + persistence ────────────────────────────────────────────────

    [Fact]
    public async Task AddAsync_PersistsEntityToDatabase()
    {
        // Arrange
        var product = new Product { Name = "New Widget", Price = 14.99m };

        // Act
        await GetRepository().AddAsync(product, CancellationToken.None);

        // The repository should call SaveChangesAsync internally,
        // OR the test calls UnitOfWork.SaveChangesAsync — adjust as needed.

        // Assert — verify via direct DB query (not the same EF context)
        var saved = await GetFromDbAsync(db => db.Products.FindAsync(product.Id).AsTask());
        saved.Should().NotBeNull();
        saved!.Name.Should().Be("New Widget");
    }
}
```

---

## Notes on Adapting This Template

**Repository scope:** Each call to `GetRepository()` resolves a fresh DI scope. This prevents EF Core identity map from serving cached entities and ensures queries hit the real database. If the repository pattern uses `UnitOfWork`, resolve `IUnitOfWork` separately and call `SaveChangesAsync` after mutations.

**EF change tracker:** When verifying persistence after a write, use `db.ChangeTracker.Clear()` in `GetFromDbAsync` to bypass the EF identity cache:
```csharp
var saved = await GetFromDbAsync(async db =>
{
    db.ChangeTracker.Clear();
    return await db.Products.FindAsync(id);
});
```

**Navigation properties:** If the repository's `GetByIdAsync` uses `.Include(p => p.Category)`, the navigation property test confirms the eager load works against a real database — something unit tests with `NSubstitute` cannot verify.

**Pagination:** For repositories with `GetPagedAsync(pageNumber, pageSize)`, test edge cases: first page, last page, empty result, page beyond total count.

**Soft delete global filter:** If `AppDbContext` configures a global query filter (`HasQueryFilter(p => !p.IsDeleted)`), the soft delete test verifies the filter is applied correctly at the DB level.
