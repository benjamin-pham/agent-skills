# Query Template Reference

## Query Record

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{OperationName}Query.cs`:

### Standard Query
```csharp
using {ProjectName}.Application.Abstractions.Messaging;

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

public sealed record {OperationName}Query(
    // Parameters for filtering/identification
    // Example: Guid BookingId
) : IQuery<{ResponseType}>;
```

### Cached Query
```csharp
using {ProjectName}.Application.Abstractions.Caching;

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

public sealed record {OperationName}Query(
    Guid {Entity}Id
) : ICachedQuery<{ResponseType}>
{
    public string CacheKey => $"{entityPlural}-{{{Entity}Id}}";
    public TimeSpan? Expiration => null; // or TimeSpan.FromMinutes(5)
}
```

**When to use ICachedQuery:**
- Data that doesn't change frequently (reference data, configuration)
- Expensive queries (complex joins, aggregations)
- High-traffic endpoints

**When to use plain IQuery:**
- Data that changes often
- Queries with user-specific authorization checks
- Simple, fast queries

---

## Query Handler

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{OperationName}QueryHandler.cs`:

### GetById Pattern
```csharp
using {ProjectName}.Application.Abstractions.Data;
using {ProjectName}.Application.Abstractions.Messaging;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.{EntityPlural};
using Dapper;
using System.Data;

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

internal sealed class {OperationName}QueryHandler : IQueryHandler<{OperationName}Query, {ResponseType}>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public {OperationName}QueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<Result<{ResponseType}>> Handle(
        {OperationName}Query request,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                id AS Id,
                name AS Name,
                description AS Description,
                created_at AS CreatedAt
            FROM {table_snake_case}
            WHERE id = @{Entity}Id
            """;

        {ResponseType}? result = await connection.QueryFirstOrDefaultAsync<{ResponseType}>(
            sql,
            new { request.{Entity}Id });

        if (result is null)
        {
            return Result.Failure<{ResponseType}>({Entity}Errors.NotFound);
        }

        return result;
    }
}
```

### GetAll / Search Pattern
```csharp
internal sealed class {OperationName}QueryHandler : IQueryHandler<{OperationName}Query, IReadOnlyList<{ResponseType}>>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public {OperationName}QueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<Result<IReadOnlyList<{ResponseType}>>> Handle(
        {OperationName}Query request,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                id AS Id,
                name AS Name,
                description AS Description,
                created_at AS CreatedAt
            FROM {table_snake_case}
            WHERE NOT EXISTS (
                -- optional: filter conditions
            )
            """;

        IEnumerable<{ResponseType}> results = await connection.QueryAsync<{ResponseType}>(
            sql,
            new
            {
                // pass query parameters
            });

        return results.ToList();
    }
}
```

### Query with Authorization
```csharp
internal sealed class {OperationName}QueryHandler : IQueryHandler<{OperationName}Query, {ResponseType}>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;
    private readonly IUserContext _userContext;

    public {OperationName}QueryHandler(
        ISqlConnectionFactory sqlConnectionFactory,
        IUserContext userContext)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
        _userContext = userContext;
    }

    public async Task<Result<{ResponseType}>> Handle(
        {OperationName}Query request,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT ...
            FROM {table_snake_case}
            WHERE id = @Id
            """;

        {ResponseType}? result = await connection.QueryFirstOrDefaultAsync<{ResponseType}>(
            sql,
            new { Id = request.{Entity}Id });

        // Resource-based authorization
        if (result is null || result.UserId != _userContext.UserId)
        {
            return Result.Failure<{ResponseType}>({Entity}Errors.NotFound);
        }

        return result;
    }
}
```

### Multi-mapping Pattern (joins)
When the response contains nested objects, use Dapper's multi-mapping:

```csharp
IEnumerable<{ResponseType}> results = await connection
    .QueryAsync<{ResponseType}, {NestedResponse}, {ResponseType}>(
        sql,
        (parent, child) =>
        {
            parent.{ChildProperty} = child;
            return parent;
        },
        new { /* parameters */ },
        splitOn: "{FirstColumnOfChildObject}");
```

---

## Response DTO

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{ResponseType}.cs`:

```csharp
namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

public sealed class {ResponseType}
{
    public Guid Id { get; init; }
    public string Name { get; init; }
    public string Description { get; init; }
    public decimal Price { get; init; }
    public DateTime CreatedAt { get; init; }
}
```

**Guidelines:**
- Use `{ get; init; }` for all properties — immutable after Dapper creates them
- `sealed class` (not record) — matches project convention
- Property names must match SQL aliases exactly (Dapper maps by name)
- For nested objects, create separate response classes (e.g., `AddressResponse`)
- If a response DTO is shared across multiple queries in the same entity, place it at the entity folder level (e.g., `{EntityPlural}/{ResponseType}.cs` instead of inside the operation subfolder)

---

## SQL Conventions for Queries

All SQL in query handlers must follow these rules:

- Table names: **snake_case plural** (e.g., `bookings`, `order_items`)
- Column names in DB: **snake_case** (e.g., `created_at`, `user_id`)
- Use SQL **aliases** to map to PascalCase DTO properties: `created_at AS CreatedAt`
- Use **raw string literals** (`"""..."""`) for SQL
- Use **parameterized queries** — never concatenate user input into SQL
- For enums stored as int, cast in SQL or let Dapper handle it
- Prefer `QueryFirstOrDefaultAsync` for single results, `QueryAsync` for lists
