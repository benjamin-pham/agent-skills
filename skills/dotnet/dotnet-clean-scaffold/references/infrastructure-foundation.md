# Infrastructure Layer — Foundation

Create all files below. Replace `{ProjectName}` with the actual project name.
For PostgreSQL use `UseNpgsql`; for SQL Server use `UseSqlServer`.

---

## src/{ProjectName}.Infrastructure/Data/AppDbContext.cs

`AppDbContext` implements `IUnitOfWork` so the Application layer can save changes
without knowing about EF Core directly.

```csharp
using Microsoft.EntityFrameworkCore;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Infrastructure.Data;

public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options), IUnitOfWork
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        return await base.SaveChangesAsync(ct);
    }
}
```

---

## src/{ProjectName}.Infrastructure/Data/Configurations/BaseEntityConfiguration.cs

All entity-specific configurations extend this class. It maps the `BaseEntity`
audit fields to snake_case columns and configures the soft-delete query filter.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Infrastructure.Data.Configurations;

public abstract class BaseEntityConfiguration<T> : IEntityTypeConfiguration<T>
    where T : BaseEntity
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
```

---

## src/{ProjectName}.Infrastructure/Repositories/Repository.cs

Generic repository implementation. Entity-specific repositories inherit from this class.

```csharp
using Microsoft.EntityFrameworkCore;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.Entities;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.Infrastructure.Repositories;

public abstract class Repository<T>(AppDbContext context) : IRepository<T>
    where T : BaseEntity
{
    protected readonly AppDbContext Context = context;

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await Context.Set<T>().FirstOrDefaultAsync(x => x.Id == id, ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default) =>
        await Context.Set<T>().ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default) =>
        await Context.Set<T>().AddAsync(entity, ct);

    public void Update(T entity) =>
        Context.Set<T>().Update(entity);

    public void Remove(T entity) =>
        Context.Set<T>().Remove(entity);
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

## src/{ProjectName}.Infrastructure/DependencyInjection.cs

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using {ProjectName}.Application.Abstractions.Data;
using {ProjectName}.Domain.Abstractions;
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

        return services;
    }
}
```
