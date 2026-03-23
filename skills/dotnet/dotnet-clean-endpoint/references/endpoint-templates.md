# Endpoint Templates

All examples use `Product` as the entity. Replace with the actual entity name and adjust properties accordingly.

## Folder Structure

```
src/{ProjectName}.API/Endpoints/
├── IEndpoint.cs
├── EndpointExtensions.cs
└── Products/
    ├── GetAllProductsEndpoint.cs
    ├── GetProductByIdEndpoint.cs
    ├── CreateProductEndpoint.cs
    ├── UpdateProductEndpoint.cs
    └── DeleteProductEndpoint.cs
```

---

## GetAll — List All Entities

`src/{ProjectName}.API/Endpoints/Products/GetAllProductsEndpoint.cs`

```csharp
using {ProjectName}.Domain.Interfaces;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.API.Endpoints.Products;

public class GetAllProductsEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/products", async (IRepository<Product> repository) =>
        {
            var products = await repository.GetAllAsync();
            return Results.Ok(products);
        })
        .WithName("GetAllProducts")
        .WithTags("Products")
        .Produces<IEnumerable<Product>>();
    }
}
```

---

## GetById — Get Single Entity

`src/{ProjectName}.API/Endpoints/Products/GetProductByIdEndpoint.cs`

```csharp
using {ProjectName}.Domain.Interfaces;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.API.Endpoints.Products;

public class GetProductByIdEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/products/{id:guid}", async (Guid id, IRepository<Product> repository) =>
        {
            var product = await repository.GetByIdAsync(id);
            return product is null
                ? Results.NotFound()
                : Results.Ok(product);
        })
        .WithName("GetProductById")
        .WithTags("Products")
        .Produces<Product>()
        .Produces(StatusCodes.Status404NotFound);
    }
}
```

---

## Create — Create New Entity

`src/{ProjectName}.API/Endpoints/Products/CreateProductEndpoint.cs`

```csharp
using {ProjectName}.Domain.Interfaces;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.API.Endpoints.Products;

public class CreateProductEndpoint : IEndpoint
{
    public record CreateProductRequest(string Name, string? Description, decimal Price, Guid CategoryId);

    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/products", async (CreateProductRequest request, IRepository<Product> repository) =>
        {
            var product = new Product
            {
                Name = request.Name,
                Description = request.Description,
                Price = request.Price,
                CategoryId = request.CategoryId
            };

            await repository.AddAsync(product);
            return Results.Created($"/api/products/{product.Id}", product);
        })
        .WithName("CreateProduct")
        .WithTags("Products")
        .Produces<Product>(StatusCodes.Status201Created)
        .Produces(StatusCodes.Status400BadRequest);
    }
}
```

---

## Update — Full Update Entity

`src/{ProjectName}.API/Endpoints/Products/UpdateProductEndpoint.cs`

```csharp
using {ProjectName}.Domain.Interfaces;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.API.Endpoints.Products;

public class UpdateProductEndpoint : IEndpoint
{
    public record UpdateProductRequest(string Name, string? Description, decimal Price, Guid CategoryId);

    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPut("/api/products/{id:guid}", async (Guid id, UpdateProductRequest request, IRepository<Product> repository) =>
        {
            var product = await repository.GetByIdAsync(id);
            if (product is null)
                return Results.NotFound();

            product.Name = request.Name;
            product.Description = request.Description;
            product.Price = request.Price;
            product.CategoryId = request.CategoryId;

            await repository.UpdateAsync(product);
            return Results.Ok(product);
        })
        .WithName("UpdateProduct")
        .WithTags("Products")
        .Produces<Product>()
        .Produces(StatusCodes.Status404NotFound)
        .Produces(StatusCodes.Status400BadRequest);
    }
}
```

---

## Delete — Soft Delete Entity

`src/{ProjectName}.API/Endpoints/Products/DeleteProductEndpoint.cs`

```csharp
using {ProjectName}.Domain.Interfaces;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.API.Endpoints.Products;

public class DeleteProductEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapDelete("/api/products/{id:guid}", async (Guid id, IRepository<Product> repository) =>
        {
            var product = await repository.GetByIdAsync(id);
            if (product is null)
                return Results.NotFound();

            await repository.DeleteAsync(product);
            return Results.NoContent();
        })
        .WithName("DeleteProduct")
        .WithTags("Products")
        .Produces(StatusCodes.Status204NoContent)
        .Produces(StatusCodes.Status404NotFound);
    }
}
```

---

## Route Naming Convention

| Entity | Route | Tag |
|--------|-------|-----|
| `Product` | `/api/products` | Products |
| `Category` | `/api/categories` | Categories |
| `OrderItem` | `/api/order-items` | Order Items |
| `UserProfile` | `/api/user-profiles` | User Profiles |

Rules:
- Plural form of the entity name
- Lowercase, kebab-case for multi-word names
- Tag uses title case with spaces

## With Entity-Specific Repository

If the entity has a dedicated repository interface (e.g., `IProductRepository` with custom methods), use that instead of the generic `IRepository<Product>`:

```csharp
app.MapGet("/api/products", async (IProductRepository repository) =>
{
    var products = await repository.GetByCategoryAsync(categoryId);
    return Results.Ok(products);
})
```
