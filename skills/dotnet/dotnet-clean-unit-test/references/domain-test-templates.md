# Domain Layer Test Templates

All examples use `Product` as the entity. Replace with the actual entity name and properties.

## Entity Test — Basic Entity

Tests for an entity that inherits `BaseEntity` (Guid key) with properties and domain behavior.

### File: `tests/{ProjectName}.Domain.Tests/Entities/ProductTests.cs`

```csharp
using {ProjectName}.Domain.Entities;
using {ProjectName}.Domain.Exceptions;

namespace {ProjectName}.Domain.Tests.Entities;

public class ProductTests
{
    [Fact]
    public void Constructor_WhenCreated_HasDefaultValues()
    {
        // Act
        var product = new Product();

        // Assert
        product.Id.Should().Be(Guid.Empty);
        product.CreatedAt.Should().Be(default);
        product.UpdatedAt.Should().Be(default);
        product.IsDeleted.Should().BeFalse();
    }

    [Fact]
    public void Properties_WhenSet_ReturnCorrectValues()
    {
        // Arrange
        var id = Guid.NewGuid();
        var name = "Test Product";
        var price = 29.99m;

        // Act
        var product = new Product
        {
            Id = id,
            Name = name,
            Price = price
        };

        // Assert
        product.Id.Should().Be(id);
        product.Name.Should().Be(name);
        product.Price.Should().Be(price);
    }

    [Fact]
    public void IsDeleted_DefaultValue_IsFalse()
    {
        // Act
        var product = new Product();

        // Assert
        product.IsDeleted.Should().BeFalse();
    }
}
```

## Entity Test — Entity with Domain Logic

When an entity has methods that enforce business rules, test those rules thoroughly.

### File: `tests/{ProjectName}.Domain.Tests/Entities/OrderTests.cs`

```csharp
using {ProjectName}.Domain.Entities;
using {ProjectName}.Domain.Exceptions;

namespace {ProjectName}.Domain.Tests.Entities;

public class OrderTests
{
    private Order CreateValidOrder() => new()
    {
        Id = Guid.NewGuid(),
        CustomerName = "John Doe",
        Status = OrderStatus.Pending
    };

    [Fact]
    public void Confirm_WhenPending_ChangesStatusToConfirmed()
    {
        // Arrange
        var order = CreateValidOrder();

        // Act
        order.Confirm();

        // Assert
        order.Status.Should().Be(OrderStatus.Confirmed);
    }

    [Fact]
    public void Confirm_WhenAlreadyConfirmed_ThrowsBadRequestException()
    {
        // Arrange
        var order = CreateValidOrder();
        order.Confirm();

        // Act
        var act = () => order.Confirm();

        // Assert
        act.Should().Throw<BadRequestException>()
            .WithMessage("*already confirmed*");
    }

    [Fact]
    public void Cancel_WhenPending_ChangesStatusToCancelled()
    {
        // Arrange
        var order = CreateValidOrder();

        // Act
        order.Cancel();

        // Assert
        order.Status.Should().Be(OrderStatus.Cancelled);
    }

    [Fact]
    public void Cancel_WhenAlreadyShipped_ThrowsBadRequestException()
    {
        // Arrange
        var order = CreateValidOrder();
        order.Confirm();
        order.Ship();

        // Act
        var act = () => order.Cancel();

        // Assert
        act.Should().Throw<BadRequestException>()
            .WithMessage("*cannot cancel*shipped*");
    }

    [Fact]
    public void AddItem_WithValidItem_IncreasesTotalAmount()
    {
        // Arrange
        var order = CreateValidOrder();
        var item = new OrderItem { ProductId = Guid.NewGuid(), Quantity = 2, UnitPrice = 15.00m };

        // Act
        order.AddItem(item);

        // Assert
        order.Items.Should().ContainSingle();
        order.TotalAmount.Should().Be(30.00m);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void AddItem_WithInvalidQuantity_ThrowsBadRequestException(int quantity)
    {
        // Arrange
        var order = CreateValidOrder();
        var item = new OrderItem { ProductId = Guid.NewGuid(), Quantity = quantity, UnitPrice = 10.00m };

        // Act
        var act = () => order.AddItem(item);

        // Assert
        act.Should().Throw<BadRequestException>()
            .WithMessage("*quantity*");
    }
}
```

## Entity Test — Entity with Custom Key Type

For entities using `BaseEntity<int>` or `BaseEntity<long>`:

```csharp
namespace {ProjectName}.Domain.Tests.Entities;

public class CategoryTests
{
    [Fact]
    public void Constructor_WhenCreated_HasDefaultIntKey()
    {
        // Act
        var category = new Category();

        // Assert
        category.Id.Should().Be(0);
    }

    [Fact]
    public void Properties_WhenSet_ReturnCorrectValues()
    {
        // Arrange & Act
        var category = new Category
        {
            Id = 42,
            Name = "Electronics",
            Description = "Electronic devices and gadgets"
        };

        // Assert
        category.Id.Should().Be(42);
        category.Name.Should().Be("Electronics");
    }
}
```

## Domain Exception Tests

### File: `tests/{ProjectName}.Domain.Tests/Exceptions/DomainExceptionTests.cs`

```csharp
using {ProjectName}.Domain.Exceptions;

namespace {ProjectName}.Domain.Tests.Exceptions;

public class NotFoundExceptionTests
{
    [Fact]
    public void Constructor_WithNameAndKey_FormatsMessage()
    {
        // Arrange
        var id = Guid.NewGuid();

        // Act
        var exception = new NotFoundException("Product", id);

        // Assert
        exception.Message.Should().Be($"Product with id '{id}' was not found.");
    }

    [Fact]
    public void Constructor_WithMessage_SetsMessage()
    {
        // Act
        var exception = new NotFoundException("Custom not found message");

        // Assert
        exception.Message.Should().Be("Custom not found message");
    }
}

public class BadRequestExceptionTests
{
    [Fact]
    public void Constructor_WithMessage_SetsMessage()
    {
        // Act
        var exception = new BadRequestException("Invalid input");

        // Assert
        exception.Message.Should().Be("Invalid input");
    }

    [Fact]
    public void Exception_InheritsFromException_IsTrue()
    {
        // Act
        var exception = new BadRequestException("test");

        // Assert
        exception.Should().BeAssignableTo<Exception>();
    }
}

public class ConflictExceptionTests
{
    [Fact]
    public void Constructor_WithMessage_SetsMessage()
    {
        // Act
        var exception = new ConflictException("Already exists");

        // Assert
        exception.Message.Should().Be("Already exists");
    }
}
```

## Adapting Templates to Real Entities

When generating tests for an actual entity in the project:

1. **Read the entity class** to discover its properties and methods
2. **Test each property** with expected values
3. **Test domain methods** — if the entity has methods like `Activate()`, `Cancel()`, `UpdatePrice(decimal)`, write tests for valid and invalid inputs
4. **Test validation** — if the entity throws exceptions for invalid state, cover each validation rule
5. **Test relationships** — if the entity has navigation properties or collection management methods (like `AddItem`), test the add/remove behavior
6. **Use `[Theory]`** for boundary conditions: zero, negative, null, empty string, max length
