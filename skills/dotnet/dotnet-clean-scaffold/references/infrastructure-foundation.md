# Infrastructure Layer — Foundation

Create all files below. Replace `{ProjectName}` with the actual project name.
For PostgreSQL use `UseNpgsql`; for SQL Server use `UseSqlServer`.

---

## src/{ProjectName}.Infrastructure/Data/AppDbContext.cs

`AppDbContext` implements `IUnitOfWork` so the Application layer can save changes
without knowing about EF Core directly.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Data;

public sealed class AppDbContext(
    DbContextOptions<AppDbContext> options,
    IDateTimeProvider dateTimeProvider,
    IUserContext userContext)
    : DbContext(options), IUnitOfWork
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var now = dateTimeProvider.UtcNow;
        string? userId = null;

        try { userId = userContext.UserId.ToString(); }
        catch { /* outside of HTTP request (migrations, seeding) */ }

        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = now;
                    entry.Entity.UpdatedBy = userId;
                    break;
            }
        }

        return await base.SaveChangesAsync(ct);
    }
}
```

---

## src/{ProjectName}.Infrastructure/Data/Configurations/BaseEntityConfiguration.cs

All entity-specific configurations extend this class. It maps the `BaseEntity`
audit fields to snake_case columns and configures the soft-delete query filter.

There are two versions:
- `BaseEntityConfiguration<T>` — shorthand for the common case (Guid key)
- `BaseEntityConfiguration<T, TKey>` — generic version for entities with non-Guid keys (e.g., `int`, `long`)

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Data.Configurations;

// Generic version — use when the entity has a non-Guid primary key
public abstract class BaseEntityConfiguration<T, TKey> : IEntityTypeConfiguration<T>
    where T : BaseEntity<TKey>
    where TKey : notnull
{
    public virtual void Configure(EntityTypeBuilder<T> builder)
    {
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Id)
            .HasColumnName("id");

        builder.Property(x => x.CreatedAt)
            .HasColumnName("created_at")
            .IsRequired();

        builder.Property(x => x.CreatedBy)
            .HasColumnName("created_by")
            .HasMaxLength(256);

        builder.Property(x => x.UpdatedAt)
            .HasColumnName("updated_at");

        builder.Property(x => x.UpdatedBy)
            .HasColumnName("updated_by")
            .HasMaxLength(256);

        builder.Property(x => x.IsDeleted)
            .HasColumnName("is_deleted")
            .IsRequired();

        builder.HasQueryFilter(x => !x.IsDeleted);
    }
}

// Shorthand for the common case — Guid primary key
public abstract class BaseEntityConfiguration<T> : BaseEntityConfiguration<T, Guid>
    where T : BaseEntity
{
}
```

---

## src/{ProjectName}.Infrastructure/Repositories/Repository.cs

Generic repository implementation. Entity-specific repositories inherit from this class.

```csharp
using Microsoft.EntityFrameworkCore;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.Infrastructure.Repositories;

public abstract class Repository<T, TKey>(AppDbContext context) : IRepository<T, TKey>
    where T : BaseEntity<TKey>
    where TKey : notnull
{
    protected readonly AppDbContext Context = context;

    public async Task<T?> GetByIdAsync(TKey id, CancellationToken ct = default) =>
        await Context.Set<T>().FindAsync([id!], ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default) =>
        await Context.Set<T>().ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default) =>
        await Context.Set<T>().AddAsync(entity, ct);

    public void Update(T entity) =>
        Context.Set<T>().Update(entity);

    public void Remove(T entity) =>
        Context.Set<T>().Remove(entity);
}

// Shorthand for the common case — Guid primary key
public abstract class Repository<T>(AppDbContext context) : Repository<T, Guid>(context)
    where T : BaseEntity
{
}
```

---

## src/{ProjectName}.Infrastructure/Data/SqlConnectionFactory.cs

Provides raw ADO.NET / Dapper connections for query-side (read) handlers.

```csharp
using System.Data;
using Npgsql;
using {ProjectName}.Application.Abstractions.Data;

namespace {ProjectName}.Infrastructure.Data;

internal sealed class SqlConnectionFactory(string connectionString) : ISqlConnectionFactory
{
    public IDbConnection CreateConnection()
    {
        var connection = new NpgsqlConnection(connectionString);
        connection.Open();
        return connection;
    }
}
```

> For **SQL Server**: replace `NpgsqlConnection` with `SqlConnection`
> and add `using Microsoft.Data.SqlClient;`.
> Add package: `dotnet add src/{ProjectName}.Infrastructure package Microsoft.Data.SqlClient`

---

## src/{ProjectName}.Infrastructure/Clock/DateTimeProvider.cs

```csharp
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Clock;

internal sealed class DateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

---

## src/{ProjectName}.Infrastructure/Caching/CacheService.cs

```csharp
using System.Buffers;
using System.Text.Json;
using Microsoft.Extensions.Caching.Distributed;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Caching;

internal sealed class CacheService(IDistributedCache distributedCache) : ICacheService
{
    public async Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default)
    {
        byte[]? bytes = await distributedCache.GetAsync(key, cancellationToken);

        return bytes is null ? default : Deserialize<T>(bytes);
    }

    public Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? expiration = null,
        CancellationToken cancellationToken = default)
    {
        byte[] bytes = Serialize(value);

        return distributedCache.SetAsync(key, bytes, CacheOptions.Create(expiration), cancellationToken);
    }

    public Task RemoveAsync(string key, CancellationToken cancellationToken = default) =>
        distributedCache.RemoveAsync(key, cancellationToken);

    private static T Deserialize<T>(byte[] bytes) =>
        JsonSerializer.Deserialize<T>(bytes)!;

    private static byte[] Serialize<T>(T value)
    {
        ArrayBufferWriter<byte> buffer = new();

        using Utf8JsonWriter writer = new(buffer);

        JsonSerializer.Serialize(writer, value);

        return buffer.WrittenSpan.ToArray();
    }
}
```

---

## src/{ProjectName}.Infrastructure/Caching/CacheOptions.cs

```csharp
using Microsoft.Extensions.Caching.Distributed;

namespace {ProjectName}.Infrastructure.Caching;

internal static class CacheOptions
{
    public static DistributedCacheEntryOptions Create(TimeSpan? expiration) =>
        expiration is null
            ? new DistributedCacheEntryOptions()
            : new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = expiration };
}
```

---

## src/{ProjectName}.Infrastructure/Authentication/UserContext.cs

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.AspNetCore.Http;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Authentication;

internal sealed class UserContext(IHttpContextAccessor contextAccessor) : IUserContext
{
    public Guid UserId =>
        contextAccessor
            .HttpContext?.User
            .GetUserId() ?? throw new ApplicationException("User context is unavailable");
}

internal static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal principal)
    {
        string? userId = principal?.FindFirstValue(JwtRegisteredClaimNames.Sub);

        return Guid.TryParse(userId, out Guid parsedUserId)
            ? parsedUserId
            : throw new ApplicationException("User identifier is unavailable");
    }
}
```

---

## src/{ProjectName}.Infrastructure/DependencyInjection.cs

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using {ProjectName}.Application.Abstractions.Data;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Infrastructure.Authentication;
using {ProjectName}.Infrastructure.Caching;
using {ProjectName}.Infrastructure.Clock;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("Database")
            ?? throw new InvalidOperationException("Connection string 'Database' is not configured.");

        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(connectionString));   // swap to UseSqlServer for SQL Server

        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());

        services.AddSingleton<ISqlConnectionFactory>(
            _ => new SqlConnectionFactory(connectionString));

        services.AddSingleton<IDateTimeProvider, DateTimeProvider>();

        services.AddHttpContextAccessor();
        services.AddScoped<IUserContext, UserContext>();

        services.AddScoped<ICacheService, CacheService>();

        return services;
    }
}
```
