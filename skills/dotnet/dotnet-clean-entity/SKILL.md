---
name: dotnet-clean-entity
description: >
  Generate rich domain entities for ASP.NET Core Clean Architecture projects.
  Produces entities with a static factory method (Create) following DDD Rich
  Domain Model patterns. Domain layer only — no EF config, repositories, or DI.
  Use whenever the user wants to create/add a new entity, model, aggregate, or
  domain object in a .NET project. Trigger on: "tạo entity", "thêm entity",
  "add entity", "create entity Product", "tạo model Order", "tạo aggregate",
  "thêm domain object", or any request about domain entities with factory
  methods or encapsulation. Even if user just says "tạo entity X", use this
  skill — all entities follow rich domain model by default.
---

# ASP.NET Core — Rich Domain Entity Generator

Generates domain entities following **Rich Domain Model** / DDD patterns:
public setters and a static factory method `Create(...)`.
This skill focuses exclusively on the **Domain layer** — no infrastructure
code is generated.

## Core Principles

1. **Public setters** — all properties use `{ get; set; }`. Encapsulation is
   enforced at the boundary through the `Create()` factory method, not through
   accessor restrictions.
2. **Static factory method `Create(...)`** — the only way to instantiate.
   Accepts all required parameters and returns a fully initialized entity.
3. **Private constructor** — prevents `new {Entity}()` from outside. Only
   `Create()` can build an instance.
4. **No half-constructed objects** — every entity starts in a valid state
   because all required fields are set inside `Create()`.

---

## Prerequisite: BaseEntity

Before generating any concrete entity, check that the base entity classes
exist in `src/{ProjectName}.Domain/Abstractions/`. If they don't, create both
classes below in the same file.

### BaseEntity\<TKey\> — generic version

Create `src/{ProjectName}.Domain/Abstractions/BaseEntity.cs`:

```csharp
namespace {ProjectName}.Domain.Abstractions;

public abstract class BaseEntity<TKey> where TKey : notnull
{
    public TKey Id { get; set; } = default!;

    public DateTime CreatedAt { get; set; }
    public string? CreatedBy { get; set; }

    public DateTime? UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }

    public bool IsDeleted { get; set; }

    protected BaseEntity()
    {
        CreatedAt = DateTime.UtcNow;
    }

    public void SoftDelete()
    {
        IsDeleted = true;
    }
}
```

### BaseEntity — Guid shorthand

Most entities use `Guid` as key. Provide a non-generic shorthand in the
**same file** (`BaseEntity.cs`), right below the generic class:

```csharp
public abstract class BaseEntity : BaseEntity<Guid>
{
    protected BaseEntity()
    {
        Id = Guid.NewGuid();
    }
}
```

### Design notes

- `BaseEntity<TKey>` supports any key type (`Guid`, `int`, `long`, `string`).
- `BaseEntity` (no type param) defaults to `Guid` and auto-generates the Id.
- `CreatedAt` is set in the constructor. `CreatedBy`, `UpdatedAt`, `UpdatedBy`
  have public setters — they are populated by infrastructure (e.g., a
  SaveChanges interceptor or EF `SaveChangesAsync` override that reads the
  current user from `IHttpContextAccessor`). The domain layer doesn't know
  about the current user.
- `UpdatedAt` is `DateTime?` (nullable) because a newly created entity has
  no update yet.
- `SoftDelete()` is the only behavior method on BaseEntity.
- Concrete entities inherit `BaseEntity` (Guid) by default. Use
  `BaseEntity<int>` or `BaseEntity<long>` when the user explicitly needs
  a non-Guid key.

---

## Workflow: Adding a New Entity

When the user asks to create an entity (e.g., "tạo entity Product"):

### Step 1 — Detect project structure

Find the `.slnx` (or `.sln`), identify `{ProjectName}`.

### Step 2 — Ensure base classes exist

Check for `BaseEntity.cs` (which should contain both `BaseEntity<TKey>` and
`BaseEntity`). Create if missing (see Prerequisite section above).

### Step 3 — Gather entity requirements

Ask the user (or infer from context) what properties the entity needs:
- What properties does this entity have?
- Which are required vs optional?
- What types? (string, decimal, int, Guid, enum, etc.)
- Are there relationships to other entities?

If the user doesn't specify, infer sensible properties from the entity name
(e.g., `Product` → Name, Description, Price).

### Step 4 — Generate entity class

Create `src/{ProjectName}.Domain/Entities/{EntityName}.cs` following this
template:

```csharp
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Domain.Entities;

public class {EntityName} : BaseEntity
{
    // ── Properties ────────────────────────────────────────
    // All properties: public set. Required props use null! initializer.
    // Optional (nullable) props don't need initializer.
    public string Name { get; set; } = null!;
    public decimal Price { get; set; }
    public string? Description { get; set; }
    public Guid CategoryId { get; set; }

    // Navigation properties (no setter — EF loads them)
    public Category? Category { get; }

    // ── Constructor (private) ─────────────────────────────
    private {EntityName}() { } // EF Core needs this

    // ── Factory Method ────────────────────────────────────
    public static {EntityName} Create(
        string name,
        decimal price,
        Guid categoryId,
        string? description = null)
    {
        return new {EntityName}
        {
            Name = name,
            Price = price,
            CategoryId = categoryId,
            Description = description
        };
    }
}
```

---

## Patterns to follow in every entity

1. **Private parameterless constructor** — `private {EntityName}() { }`.
   Required for EF Core materialization. Never called by application code.

2. **`Create(...)` factory method** — accepts all required parameters first,
   then optional parameters with defaults. Returns the entity via object
   initializer. Required parameters should NOT have default values.

3. **Property access levels:**
   - Scalar properties: `{ get; set; }`
   - Navigation references: `{ get; }` only (no setter, EF loads them)
   - Collection navigations: private backing field + public read-only accessor:

     ```csharp
     private readonly List<OrderItem> _items = [];
     public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
     ```

4. **Required vs optional parameters in `Create()`:**
   - Required string props → `string name` (no default)
   - Optional string props → `string? description = null`
   - Required FK → `Guid categoryId` (no default)
   - Enums with a default state → set inside `Create()` body, not as parameter

   Example with an enum:
   ```csharp
   public static Order Create(string customerName, decimal totalAmount)
   {
       return new Order
       {
           CustomerName = customerName,
           TotalAmount = totalAmount,
           Status = OrderStatus.Pending  // default state, not a parameter
       };
   }
   ```

---

## Full Example: Order with State Transitions

This example shows a complete entity with an enum status and methods that
transition between states. Use this as a reference when generating entities
that have state/lifecycle management.

`src/{ProjectName}.Domain/Enums/OrderStatus.cs`:

```csharp
namespace {ProjectName}.Domain.Enums;

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}
```

`src/{ProjectName}.Domain/Entities/Order.cs`:

```csharp
using {ProjectName}.Domain.Abstractions;
using {ProjectName}.Domain.Enums;

namespace {ProjectName}.Domain.Entities;

public class Order : BaseEntity
{
    // ── Properties ────────────────────────────────────────
    public string CustomerName { get; set; } = null!;
    public string ShippingAddress { get; set; } = null!;
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public string? Note { get; set; }

    // ── Collection Navigation ─────────────────────────────
    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    // ── Constructor (private) ─────────────────────────────
    private Order() { }

    // ── Factory Method ────────────────────────────────────
    public static Order Create(
        string customerName,
        string shippingAddress,
        decimal totalAmount,
        string? note = null)
    {
        return new Order
        {
            CustomerName = customerName,
            ShippingAddress = shippingAddress,
            TotalAmount = totalAmount,
            Status = OrderStatus.Pending,
            Note = note
        };
    }

    // ── State Transitions ─────────────────────────────────
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException(
                $"Cannot confirm order in '{Status}' status. Only Pending orders can be confirmed.");

        Status = OrderStatus.Confirmed;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException(
                $"Cannot ship order in '{Status}' status. Only Confirmed orders can be shipped.");

        Status = OrderStatus.Shipped;
    }

    public void Deliver()
    {
        if (Status != OrderStatus.Shipped)
            throw new InvalidOperationException(
                $"Cannot deliver order in '{Status}' status. Only Shipped orders can be delivered.");

        Status = OrderStatus.Delivered;
    }

    public void Cancel()
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new InvalidOperationException(
                $"Cannot cancel order in '{Status}' status.");

        Status = OrderStatus.Cancelled;
    }
}
```

Key patterns in this example:
- `Status` starts as `Pending` inside `Create()` — the caller doesn't choose it.
- Each transition method checks the current state before allowing the change.
- The transition methods throw `InvalidOperationException` with a clear
  message explaining why the transition is not allowed.
- The entity controls its own state machine through domain methods.

---

## Entity Naming

- If the user provides the entity name in Vietnamese, translate to English
  PascalCase and confirm with the user.
- Class name: `PascalCase` singular (e.g., `Product`, `OrderItem`).

---

## Important Reminders

- Use **file-scoped namespaces** everywhere.
- Use **C# 13** features where appropriate (collection expressions `[]`, etc.)
  but NOT primary constructors on entities — entities need a private
  parameterless constructor for EF Core.
- This skill generates **Domain layer only**. Do NOT create EF configurations,
  repositories, DbSet registrations, or DI wiring.
- When the entity has relationships, define navigation properties and FK
  `Guid` properties in the entity, but leave EF relationship configuration
  to the infrastructure layer.
- If the entity has an enum (e.g., `OrderStatus`), define it in a separate
  file at `src/{ProjectName}.Domain/Enums/{EnumName}.cs`.
