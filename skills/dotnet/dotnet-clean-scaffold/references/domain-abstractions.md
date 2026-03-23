# Domain Layer — Abstractions & Base Classes

Create all files below. Replace `{ProjectName}` with the actual project name.

---

## src/{ProjectName}.Domain/Abstractions/BaseEntity.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public abstract class BaseEntity<TKey> where TKey : notnull
{
    public TKey Id { get; init; } = default!;

    public DateTime CreatedAt { get; set; }
    public string? CreatedBy { get; set; }

    public DateTime? UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }

    public bool IsDeleted { get; set; }

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
namespace {ProjectName}.Domain.Abstractions;

public interface IRepository<T, TKey>
    where T : BaseEntity<TKey>
    where TKey : notnull
{
    Task<T?> GetByIdAsync(TKey id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
}

// Shorthand for the common case — Guid primary key
public interface IRepository<T> : IRepository<T, Guid>
    where T : BaseEntity
{
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

## src/{ProjectName}.Domain/Abstractions/IDateTimeProvider.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}
```

---

## src/{ProjectName}.Domain/Abstractions/IUserContext.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public interface IUserContext
{
    Guid UserId { get; }
}
```

---

## src/{ProjectName}.Domain/Abstractions/ICacheService.cs

```csharp
namespace {ProjectName}.Domain.Abstractions;

public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default);

    Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? expiration = null,
        CancellationToken cancellationToken = default);

    Task RemoveAsync(string key, CancellationToken cancellationToken = default);
}
```
