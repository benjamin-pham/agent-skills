# Command Template Reference

## Command Record

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{OperationName}Command.cs`:

```csharp
using {ProjectName}.Application.Abstractions.Messaging;

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

public sealed record {OperationName}Command(
    // Properties based on user requirements
    // Example for CreateProduct:
    // string Name,
    // decimal Price,
    // string? Description
) : ICommand<{ResponseType}>;
```

**Guidelines:**
- Use `ICommand` (no generic) if the command returns nothing (void → `Result`)
- Use `ICommand<Guid>` for create operations that return the new entity's ID
- Use `ICommand<TResponse>` for operations returning a custom DTO
- Properties are positional parameters in the record — keep them simple value types
- Do NOT put domain objects as command properties — use primitive types or simple DTOs

---

## Command Handler

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{OperationName}CommandHandler.cs`:

```csharp
using {ProjectName}.Application.Abstractions.Messaging;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.{EntityPlural};

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

internal sealed class {OperationName}CommandHandler : ICommandHandler<{OperationName}Command, {ResponseType}>
{
    private readonly I{Entity}Repository _{entityCamel}Repository;
    private readonly IUnitOfWork _unitOfWork;

    public {OperationName}CommandHandler(
        I{Entity}Repository {entityCamel}Repository,
        IUnitOfWork unitOfWork)
    {
        _{entityCamel}Repository = {entityCamel}Repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<{ResponseType}>> Handle(
        {OperationName}Command request,
        CancellationToken cancellationToken)
    {
        // 1. Load entity/entities from repository
        // 2. Perform domain logic (call domain methods, create new entities)
        // 3. Persist via repository + UnitOfWork
        // 4. Return Result.Success or Result.Failure

        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return entity.Id; // or Result.Success() for void commands
    }
}
```

**Handler patterns by operation type:**

### Create
```csharp
public async Task<Result<Guid>> Handle(Create{Entity}Command request, CancellationToken cancellationToken)
{
    // Use domain factory method if available
    var entity = {Entity}.Create(/* map request properties */);

    _{entityCamel}Repository.Add(entity);
    await _unitOfWork.SaveChangesAsync(cancellationToken);

    return entity.Id;
}
```

### Update
```csharp
public async Task<Result> Handle(Update{Entity}Command request, CancellationToken cancellationToken)
{
    {Entity} entity = await _{entityCamel}Repository.GetByIdAsync(request.Id, cancellationToken);

    if (entity is null)
    {
        return Result.Failure({Entity}Errors.NotFound);
    }

    // Call domain methods to update
    // entity.UpdateDetails(request.Name, request.Description);

    await _unitOfWork.SaveChangesAsync(cancellationToken);

    return Result.Success();
}
```

### Delete
```csharp
public async Task<Result> Handle(Delete{Entity}Command request, CancellationToken cancellationToken)
{
    {Entity} entity = await _{entityCamel}Repository.GetByIdAsync(request.Id, cancellationToken);

    if (entity is null)
    {
        return Result.Failure({Entity}Errors.NotFound);
    }

    // Soft delete via domain method or repository
    _{entityCamel}Repository.Remove(entity);
    await _unitOfWork.SaveChangesAsync(cancellationToken);

    return Result.Success();
}
```

**Key rules:**
- Always check if entity exists before update/delete — return `Result.Failure` with domain error
- Use domain factory methods (`Entity.Create(...)`) instead of `new Entity()`
- Inject only what's needed — don't inject services "just in case"
- Wrap risky operations in try/catch only when specific exceptions are expected (e.g., `ConcurrencyException`)
- Call `_unitOfWork.SaveChangesAsync()` — NOT the repository's save method

---

## Command Validator

Create `src/{ProjectName}.Application/{EntityPlural}/{OperationName}/{OperationName}CommandValidator.cs`:

```csharp
using FluentValidation;

namespace {ProjectName}.Application.{EntityPlural}.{OperationName};

internal class {OperationName}CommandValidator : AbstractValidator<{OperationName}Command>
{
    public {OperationName}CommandValidator()
    {
        // Validate required fields
        RuleFor(x => x.PropertyName).NotEmpty();

        // Validate string lengths
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);

        // Validate email format
        RuleFor(x => x.Email).EmailAddress();

        // Validate numeric ranges
        RuleFor(x => x.Price).GreaterThan(0);

        // Validate date logic
        RuleFor(x => x.StartDate).LessThan(x => x.EndDate);

        // Validate password strength
        RuleFor(x => x.Password).NotEmpty().MinimumLength(5);

        // Validate Guid not empty
        RuleFor(x => x.EntityId).NotEmpty();
    }
}
```

**Guidelines:**
- Validator is `internal class` (not sealed) — matches project convention
- Only validate **shape and format** — not business rules that need DB access
- Business rule validation belongs in the Handler (e.g., "does this entity exist?")
- Every command property that's required should have at least `NotEmpty()`
- String properties should have `MaximumLength()` matching the DB column constraint
- The `ValidationBehavior` pipeline will run validators automatically before the handler


