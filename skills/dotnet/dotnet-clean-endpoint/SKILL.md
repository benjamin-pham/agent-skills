---
name: dotnet-clean-endpoint
description: >
  Generate minimal API endpoints for ASP.NET Core Clean Architecture projects
  using the class-per-endpoint pattern. Each endpoint is a class implementing
  IEndpoint, organized in Endpoints/{Entity}/ folders. Covers CRUD generation,
  auto-registration via assembly scanning, request/response DTOs, and route
  conventions. Trigger whenever the user wants to add an endpoint, create an API,
  add CRUD for an entity, or wire up a new route — including Vietnamese like
  "tạo endpoint", "thêm API cho entity", "tạo CRUD", "thêm route".
  Do NOT trigger for Serilog setup, entity generation, or project scaffolding.
---

# Minimal API Endpoints — Class-per-Endpoint Pattern

Generates endpoints using a structured pattern where each HTTP operation lives in its own class. This keeps endpoint logic focused, testable, and easy to find — instead of one giant static class with all routes for an entity.

## Step 1 — IEndpoint Infrastructure (one-time setup)

Read `references/endpoint-infrastructure.md` for the complete code. Create these files if they don't exist yet:

1. **`IEndpoint` interface** — `src/{ProjectName}.API/Endpoints/IEndpoint.cs`
2. **`EndpointExtensions`** — `src/{ProjectName}.API/Endpoints/EndpointExtensions.cs`
   - Scans the assembly for all classes implementing `IEndpoint`
   - Creates an instance of each and calls `MapEndpoint(app)`
3. **Update `Program.cs`** — replace manual endpoint mapping with `app.MapEndpoints()`

This is a one-time setup. Once the infrastructure exists, skip to Step 2 for each new entity.

## Step 2 — Generate Endpoint Classes

For each entity, create endpoint classes in `src/{ProjectName}.API/Endpoints/{Entity}/`. Each class handles exactly one HTTP operation.

Read `references/endpoint-templates.md` for the complete CRUD templates. The standard set for an entity:

| Class | Route | HTTP Method | Purpose |
|-------|-------|-------------|---------|
| `GetAll{Entity}Endpoint` | `GET /api/{entities}` | GET | List all (with optional filtering) |
| `Get{Entity}ByIdEndpoint` | `GET /api/{entities}/{id}` | GET | Get single by ID |
| `Create{Entity}Endpoint` | `POST /api/{entities}` | POST | Create new |
| `Update{Entity}Endpoint` | `PUT /api/{entities}/{id}` | PUT | Full update |
| `Delete{Entity}Endpoint` | `DELETE /api/{entities}/{id}` | DELETE | Soft delete |

Key patterns:
- Class name ends with `Endpoint` — this is a convention, not enforced by the interface
- Each class implements `IEndpoint` and has one method: `MapEndpoint`
- Dependencies (repositories, services, loggers) come through **delegate parameters**, not constructor injection — the class itself is only instantiated once during registration
- Use `MapGroup` to set the base route and tags, then map the single operation on that group
- Route names follow REST conventions: plural, kebab-case for multi-word entities (`/api/order-items`)

## Step 3 — Request / Response DTOs

For POST and PUT endpoints, create request DTOs. For GET endpoints returning projected data, create response DTOs.

Place DTOs as nested records inside the endpoint class when they are specific to that endpoint, or in a shared location if reused:

```csharp
public class CreateProductEndpoint : IEndpoint
{
    public record CreateProductRequest(string Name, string? Description, decimal Price, Guid CategoryId);

    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        // ...
    }
}
```

If DTOs are shared across multiple endpoints (e.g., a `ProductResponse` used by both Get endpoints), place them in `src/{ProjectName}.API/Endpoints/{Entity}/{Entity}Dtos.cs`.

## Step 4 — Validate

1. Verify all endpoint files are created in the correct folders
2. Verify `IEndpoint` and `EndpointExtensions` exist
3. Verify `Program.cs` calls `app.MapEndpoints()`
4. Run `dotnet build` to confirm compilation

## Important Notes

- **One class = one operation.** Don't put multiple MapGet/MapPost in one class. The whole point is separation.
- **Delegate parameters for DI.** The endpoint class is instantiated once at startup for registration. Runtime dependencies (repos, services) go in the lambda parameters inside `MapEndpoint`, where the DI container resolves them per-request.
- **Route conventions**: `/api/{entities}` — plural, lowercase. Multi-word entities use kebab-case (`/api/order-items` for `OrderItem`).
- **OpenAPI metadata**: use `.WithName()`, `.WithTags()`, `.Produces<T>()` to enrich Scalar/OpenAPI docs.
- **Soft delete**: the Delete endpoint sets `IsDeleted = true` via the repository, not a hard delete.
- Target **.NET 10** with **C# 13** features (file-scoped namespaces, primary constructors).
