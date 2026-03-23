# Domain Layer — Abstractions & Base Classes

Create all files below. Replace `{ProjectName}` with the actual project name.

---

## src/{ProjectName}.Domain/Entities/BaseEntity.cs

```csharp
namespace {ProjectName}.Domain.Entities;

public abstract class BaseEntity<TKey> where TKey : notnull
{
    public TKey Id { get; init; } = default!;

    public DateTime CreatedAt { get; set; }
    public string? CreatedBy { get; set; }

    public DateTime? UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }

    public bool IsDeleted { get; private set; }

    protected BaseEntity()
    {
        CreatedAt = DateTime.UtcNow;
    }

    public void SoftDelete() => IsDeleted = true;
}

public abstract class BaseEntity : BaseEntity<Guid>
{
    protected BaseEntity()
    {
        Id = Guid.NewGuid();
    }
}
```

---

## src/{ProjectName}.Domain/Abstractions/IRepository.cs

```csharp
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Domain.Abstractions;

public interface IRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
}
```

---

## src/{ProjectName}.Domain/Abstractions/IUnitOfWork.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

---

## src/{ProjectName}.Domain/Abstractions/Error.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public record Error(string Code, string Description)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static readonly Error NullValue = new("General.Null", "A null value was provided.");

    public static Error NotFound(string entity, object id) =>
        new($"{entity}.NotFound", $"The {entity} with Id '{id}' was not found.");

    public static Error Conflict(string entity) =>
        new($"{entity}.Conflict", $"The {entity} already exists.");

    public static Error Validation(string message) =>
        new("General.Validation", message);
}
```

---

## src/{ProjectName}.Domain/Abstractions/Result.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);

    public static Result<TValue> Success<TValue>(TValue value) =>
        new(value, true, Error.None);

    public static Result<TValue> Failure<TValue>(Error error) =>
        new(default, false, error);
}

public class Result<TValue> : Result
{
    private readonly TValue? _value;

    internal Result(TValue? value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        _value = value;
    }

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("Cannot access the value of a failed result.");

    public static implicit operator Result<TValue>(TValue value) =>
        Result.Success(value);
}
```

---

## src/{ProjectName}.Domain/Abstractions/ICommand.cs

```csharp
using MediatR;

namespace {ProjectName}.Domain.Abstractions;

public interface ICommand : IRequest<Result>
{
}

public interface ICommand<TResponse> : IRequest<Result<TResponse>>
{
}
```

---

## src/{ProjectName}.Domain/Abstractions/IQuery.cs

```csharp
using MediatR;

namespace {ProjectName}.Domain.Abstractions;

public interface IQuery<TResponse> : IRequest<Result<TResponse>>
{
}
```

> **Note:** ICommand and IQuery reference MediatR, so the Domain project needs a reference to
> `{ProjectName}.Application` OR you move ICommand/IQuery into the Application layer.
> The recommended approach: keep them in Domain and add a MediatR NuGet package to Domain
> (`dotnet add src/{ProjectName}.Domain package MediatR`), or move them to Application.
> Either is fine — just be consistent. The simplest option is to add MediatR to Domain.
