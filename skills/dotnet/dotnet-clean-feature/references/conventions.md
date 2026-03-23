# Conventions Reference

## C# Code Style

| Rule | Convention |
|---|---|
| Namespaces | File-scoped (`namespace X;`) |
| Handlers | `internal sealed class` |
| Validators | `internal class` (not sealed) |
| Commands & Queries | `public sealed record` |
| Response DTOs | `public sealed class` with `{ get; init; }` |
| Constructor DI in Handlers | Explicit constructor + `private readonly` fields |
| String defaults | `string.Empty` (not `""`) |
| Null checks | Pattern matching (`entity is null`, `entity is not null`) |
| Collections | `IReadOnlyList<T>` for return types |
| Cancellation | Always pass `CancellationToken cancellationToken` |

## CQRS Separation

### Write Side (Commands)
- Use **EF Core repositories** (`I{Entity}Repository`)
- Use **IUnitOfWork** for transactional persistence
- Repository methods: `GetByIdAsync`, `Add`, `Remove`
- Call `_unitOfWork.SaveChangesAsync()` — NOT repository save

### Read Side (Queries)
- Use **Dapper** via `ISqlConnectionFactory`
- Write **raw SQL** — do NOT use EF Core for reads
- Map DB columns → DTO properties via SQL aliases
- No repository injection in query handlers

## Naming Conventions

### Operations
- Use **domain language** over generic CRUD:
  - `ReserveBooking` instead of `CreateBooking`
  - `SearchApartments` instead of `GetAllApartments`
  - `RegisterUser` instead of `CreateUser`
- When domain language isn't obvious, generic CRUD is fine:
  - `CreateProduct`, `UpdateProduct`, `DeleteProduct`
  - `GetProduct`, `GetAllProducts`

### Files and Folders
```
{EntityPlural}/
  {OperationName}/
    README.md
    {OperationName}Command.cs
    {OperationName}CommandHandler.cs
    {OperationName}CommandValidator.cs
    {OperationName}Query.cs
    {OperationName}QueryHandler.cs
    {ResponseName}.cs
```

### Response DTOs
- `{Entity}Response` for single entity (e.g., `BookingResponse`)
- `{Entity}ListItem` or reuse `{Entity}Response` for list items
- Nested DTOs: `{Child}Response` (e.g., `AddressResponse`)

## Error Handling

- Handlers return `Result<T>` — never throw exceptions for business logic failures
- Use `{Entity}Errors` static class in Domain layer for error definitions:
  ```csharp
  // In Domain/{EntityPlural}/{Entity}Errors.cs
  public static class {Entity}Errors
  {
      public static Error NotFound = new("{Entity}.NotFound", "The {entity} was not found");
      public static Error Overlap = new("{Entity}.Overlap", "The {entity} overlaps with an existing one");
  }
  ```
- Return `Result.Failure<T>({Entity}Errors.NotFound)` when entity not found
- Catch specific exceptions only when expected (e.g., `ConcurrencyException`)
- Validation errors are thrown by `ValidationBehavior` as `ValidationException` — NOT handled in the handler

## SQL Style in Query Handlers

```sql
-- Use raw string literals
const string sql = """
    SELECT
        id AS Id,
        first_name AS FirstName,
        last_name AS LastName,
        email AS Email,
        created_at AS CreatedAt
    FROM users
    WHERE id = @UserId
    """;
```

- **Indentation**: 4 spaces inside the raw string
- **Aliases**: Every column gets an explicit `AS PascalCaseAlias`
- **Parameters**: Use `@ParameterName` matching the anonymous object property
- **Table references**: snake_case plural (e.g., `users`, `bookings`, `order_items`)
- **Column references**: snake_case (e.g., `first_name`, `created_at`)
- Follow `dotnet-clean-entity` skill's `references/snake_case_rules.md` for naming

## Dependencies by File Type

| File Type | Typical Dependencies |
|---|---|
| Command | `{ProjectName}.Application.Abstractions.Messaging` |
| Command Handler | Messaging, `Domain.Abstractions`, entity repositories, `IUnitOfWork` |
| Command Validator | `FluentValidation` |
| Query | `{ProjectName}.Application.Abstractions.Messaging` (or `.Caching`) |
| Query Handler | Messaging, `Domain.Abstractions`, `Abstractions.Data`, `Dapper`, `System.Data` |
| Response DTO | Own namespace only |
