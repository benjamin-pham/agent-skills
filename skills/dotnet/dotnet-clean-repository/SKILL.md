---
name: dotnet-clean-repository
description: >
  Generate EF Core infrastructure for entities in an ASP.NET Core Clean
  Architecture project. Produces: Fluent API configuration with snake_case
  naming, repository interface + implementation, DbSet registration, and
  DI wiring. Use whenever the user wants to add EF Core config,
  repository, or persistence layer for a domain entity. Trigger on: "tạo
  repository", "thêm repository", "add repository", "tạo EF config",
  "tạo configuration", "add DbSet", "setup persistence", "cấu hình EF Core",
  or any Vietnamese/English request about EF Core configuration, repository
  pattern, or database mapping in a .NET clean architecture project.
metadata:
  related-skills:
    - dotnet-clean-architect
---

# ASP.NET Core — EF Core & Repository Generator

Generates the persistence/infrastructure layer for domain entities: EF Core
Fluent API configurations (snake_case), repository interfaces, generic
repository implementation, and DI registration.

This skill handles everything in the **Infrastructure layer** + repository
interfaces in the **Domain layer**.

---

## Workflow

When the user asks to add repository / EF config for an entity
(e.g., "tạo repository cho Product"):

1. Detect project structure (find `.slnx` / `.sln`, identify `{ProjectName}`)
2. Verify base infrastructure exists (`IRepository<T>`, `Repository<T>`,
   `BaseEntityConfiguration<T>`, `AppDbContext`)
   - If exists → skip straight to entity-specific files
3. Generate entity-specific files (Configuration, Repository, DbSet, DI)

---

## Prerequisites

This skill assumes the following already exist in the project (typically
created during project setup):

- `IRepository<T>` — generic repository interface in Domain layer
- `BaseEntityConfiguration<T>` — abstract EF config mapping `BaseEntity`
  fields to snake_case columns, with `HasQueryFilter(x => !x.IsDeleted)`
- `Repository<T>` — generic repository implementation in Infrastructure layer
- `AppDbContext` — EF Core DbContext

If any of these are missing, create them before running this skill.

---

## Entity-Specific Files

When adding infrastructure for a concrete entity (e.g., `Product`):

### 1 — EF Core Configuration

Create `src/{ProjectName}.Infrastructure/Data/Configurations/{EntityName}Configuration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Infrastructure.Data.Configurations;

public class {EntityName}Configuration : BaseEntityConfiguration<{EntityName}>
{
    public override void Configure(EntityTypeBuilder<{EntityName}> builder)
    {
        base.Configure(builder);

        builder.ToTable("{snake_case_plural_table_name}");

        // Map each property to snake_case column name
        // builder.Property(x => x.Name).HasColumnName("name").HasMaxLength(200).IsRequired();
        // builder.Property(x => x.Price).HasColumnName("price").HasColumnType("decimal(18,2)");

        // Foreign keys
        // builder.HasOne(x => x.Category)
        //     .WithMany(x => x.Products)
        //     .HasForeignKey(x => x.CategoryId)
        //     .HasConstraintName("fk_{table}_{column}");

        // Indexes
        // builder.HasIndex(x => x.Name).HasDatabaseName("ix_{table}_name");
    }
}
```

**Critical:** Follow `references/snake_case_rules.md` strictly:
- Table: plural snake_case (`Product` → `products`, `OrderItem` → `order_items`)
- Columns: snake_case (`CategoryId` → `category_id`)
- FK constraints: `fk_{table}_{column}`
- Indexes: `ix_{table}_{columns}`

Always call `base.Configure(builder)` first — it maps `BaseEntity` fields.

### 2 — Repository Interface (Domain Layer)

Create `src/{ProjectName}.Domain/Abstractions/I{EntityName}Repository.cs`:

```csharp
namespace {ProjectName}.Domain.Abstractions;

// Most entities use Guid key — extend IRepository<T> (shorthand)
public interface I{EntityName}Repository : IRepository<{EntityName}>
{
    // Add entity-specific query methods here if needed
    // Task<IReadOnlyList<{EntityName}>> GetByCategoryAsync(Guid categoryId, CancellationToken ct = default);
}

// If the entity uses a non-Guid key (e.g., int), use the generic form instead:
// public interface I{EntityName}Repository : IRepository<{EntityName}, int> { }
```

### 3 — Repository Implementation

Create `src/{ProjectName}.Infrastructure/Repositories/{EntityName}Repository.cs`:

```csharp
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.Entities;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.Infrastructure.Repositories;

// Most entities — Guid key, extend Repository<T> (shorthand)
public class {EntityName}Repository(AppDbContext context)
    : Repository<{EntityName}>(context), I{EntityName}Repository
{
    // Implement entity-specific query methods here
}

// Non-Guid key (e.g., int) — use the generic form instead:
// public class {EntityName}Repository(AppDbContext context)
//     : Repository<{EntityName}, int>(context), I{EntityName}Repository { }
```

### 4 — Register DbSet in AppDbContext

Add to `AppDbContext.cs`:

```csharp
public DbSet<{EntityName}> {EntityNamePlural} => Set<{EntityName}>();
```

### 5 — Register Repository in DI

Add to `src/{ProjectName}.Infrastructure/DependencyInjection.cs`:

```csharp
services.AddScoped<I{EntityName}Repository, {EntityName}Repository>();
```

---

## Full Example: Order

Given an `Order` entity with `OrderStatus` enum, here's the complete
infrastructure output:

`src/{ProjectName}.Infrastructure/Data/Configurations/OrderConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Infrastructure.Data.Configurations;

public class OrderConfiguration : BaseEntityConfiguration<Order>
{
    public override void Configure(EntityTypeBuilder<Order> builder)
    {
        base.Configure(builder);

        builder.ToTable("orders");

        builder.Property(x => x.CustomerName)
            .HasColumnName("customer_name")
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.ShippingAddress)
            .HasColumnName("shipping_address")
            .HasMaxLength(500)
            .IsRequired();

        builder.Property(x => x.TotalAmount)
            .HasColumnName("total_amount")
            .HasColumnType("decimal(18,2)");

        builder.Property(x => x.Status)
            .HasColumnName("status")
            .HasConversion<string>()
            .HasMaxLength(50);

        builder.Property(x => x.Note)
            .HasColumnName("note")
            .HasMaxLength(1000);

        builder.HasMany(x => x.Items)
            .WithOne()
            .HasForeignKey("order_id")
            .HasConstraintName("fk_order_items_order_id");

        builder.HasIndex(x => x.Status).HasDatabaseName("ix_orders_status");
    }
}
```

Key patterns in the example:
- Enum `Status` uses `.HasConversion<string>()` to store as text in DB.
- Collection navigation `Items` configured with FK constraint naming.
- `base.Configure(builder)` is called first to map BaseEntity fields.

---

## Important Reminders

- Use **file-scoped namespaces** everywhere.
- Use **C# 13** features: primary constructors, collection expressions.
- Always call `base.Configure(builder)` in entity configurations.
- Follow **snake_case** strictly for all DB objects — read
  `references/snake_case_rules.md` when unsure.
- All entity properties must be mapped with `HasColumnName("snake_case")`.
- When adding relationships, configure **both sides** of the navigation and
  set proper cascade behavior.
- If user provides entity name in Vietnamese, translate to English PascalCase
  and confirm with the user.
