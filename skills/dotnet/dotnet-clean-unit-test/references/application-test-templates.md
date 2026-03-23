# Application Layer Test Templates

The Application layer uses **MediatR** for CQRS (Commands and Queries) and **FluentValidation** for input validation. Tests use **NSubstitute** to mock dependencies so each test focuses on a single step in the handler's pipeline.

All examples use product/booking concepts. Replace with actual class names and adapt to the project's handlers.

---

## Core Pattern: Test Each Step Separately

The key insight for handler tests is to **test each guard clause in isolation**. A handler typically:
1. Fetches resource A — fails if not found
2. Fetches resource B — fails if not found
3. Checks business rule — fails if violated
4. Saves — fails if concurrent conflict
5. Returns success

Write one test per step. Each test only sets up the steps before it as successes, then sets up the current step to fail (or succeed for the happy path).

---

## Folder Structure — One Handler = One Test File

Each handler gets its own dedicated test file. Never combine tests for multiple handlers in one file.

```
src/.../Commands/CreateProductCommandHandler.cs
  → tests/.../Commands/CreateProductCommandHandlerTests.cs

src/.../Commands/UpdateProductCommandHandler.cs
  → tests/.../Commands/UpdateProductCommandHandlerTests.cs

src/.../Queries/GetAllProductsQueryHandler.cs
  → tests/.../Queries/GetAllProductsQueryHandlerTests.cs
```

Full test project layout:

```
tests/{ProjectName}.Application.Tests/
├── GlobalUsings.cs
├── TestData/                              ← Static factory helpers (one file per entity)
│   ├── ProductData.cs
│   └── CategoryData.cs
└── Features/
    └── Products/
        ├── Queries/
        │   ├── GetAllProductsQueryHandlerTests.cs   ← one handler, one file
        │   └── GetProductByIdQueryHandlerTests.cs   ← one handler, one file
        ├── Commands/
        │   ├── CreateProductCommandHandlerTests.cs  ← one handler, one file
        │   ├── UpdateProductCommandHandlerTests.cs  ← one handler, one file
        │   └── DeleteProductCommandHandlerTests.cs  ← one handler, one file
        └── Validators/
            └── CreateProductCommandValidatorTests.cs
```

---

## Test Data Factories

Put static factory methods in a `TestData/` folder — one class per entity. This avoids scattering `new Product { ... }` across every test and makes it easy to create valid objects in one line.

### File: `tests/{ProjectName}.Application.Tests/TestData/ProductData.cs`

```csharp
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Application.Tests.TestData;

public static class ProductData
{
    public static Product Create(
        Guid? id = null,
        string name = "Test Product",
        decimal price = 9.99m,
        string? description = null,
        Guid? categoryId = null)
    {
        return new Product
        {
            Id = id ?? Guid.NewGuid(),
            Name = name,
            Price = price,
            Description = description,
            CategoryId = categoryId ?? Guid.NewGuid()
        };
    }
}
```

```csharp
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Application.Tests.TestData;

public static class CategoryData
{
    public static Category Create(Guid? id = null, string name = "Test Category")
    {
        return new Category
        {
            Id = id ?? Guid.NewGuid(),
            Name = name
        };
    }
}
```

---

## Command Handler Tests — Step-by-Step Pattern

Each test covers exactly one point of failure. Later tests build up successful context for all prior steps before testing their own failure point. This makes each test's intent immediately obvious.

### File: `tests/{ProjectName}.Application.Tests/Features/Products/Commands/CreateProductCommandHandlerTests.cs`

```csharp
using {ProjectName}.Application.Features.Products.Commands;
using {ProjectName}.Application.Tests.TestData;
using {ProjectName}.Domain.Entities;
using {ProjectName}.Domain.Exceptions;
using {ProjectName}.Domain.Interfaces;

namespace {ProjectName}.Application.Tests.Features.Products.Commands;

public class CreateProductCommandHandlerTests
{
    // Static shared command — defined once at class level, reused across all tests
    private static readonly CreateProductCommand Command = new(
        Name: "Widget",
        Description: "A nice widget",
        Price: 9.99m,
        CategoryId: Guid.NewGuid());

    private readonly IProductRepository _productRepository;
    private readonly ICategoryRepository _categoryRepository;
    private readonly CreateProductCommandHandler _sut;

    public CreateProductCommandHandlerTests()
    {
        _productRepository = Substitute.For<IProductRepository>();
        _categoryRepository = Substitute.For<ICategoryRepository>();
        _sut = new CreateProductCommandHandler(_productRepository, _categoryRepository);
    }

    // Step 1: Category must exist
    [Fact]
    public async Task Handle_WhenCategoryNotFound_ThrowsNotFoundException()
    {
        // Arrange
        _categoryRepository
            .GetByIdAsync(Command.CategoryId, Arg.Any<CancellationToken>())
            .Returns((Category?)null);

        // Act
        var act = async () => await _sut.Handle(Command, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<NotFoundException>();
        await _productRepository.DidNotReceive().AddAsync(
            Arg.Any<Product>(), Arg.Any<CancellationToken>());
    }

    // Step 2 (happy path): Category exists → product is created
    [Fact]
    public async Task Handle_WhenCategoryExists_CreatesProductAndReturnsId()
    {
        // Arrange
        var category = CategoryData.Create(Command.CategoryId);
        _categoryRepository
            .GetByIdAsync(Command.CategoryId, Arg.Any<CancellationToken>())
            .Returns(category);

        _productRepository
            .AddAsync(Arg.Any<Product>(), Arg.Any<CancellationToken>())
            .Returns(callInfo =>
            {
                var p = callInfo.Arg<Product>();
                p.Id = Guid.NewGuid();
                return p;
            });

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.Should().NotBe(Guid.Empty);
    }

    // Step 2 (verify interaction): Product saved with correct data
    [Fact]
    public async Task Handle_WhenCategoryExists_SavesProductWithCorrectData()
    {
        // Arrange
        var category = CategoryData.Create(Command.CategoryId);
        _categoryRepository
            .GetByIdAsync(Command.CategoryId, Arg.Any<CancellationToken>())
            .Returns(category);

        _productRepository
            .AddAsync(Arg.Any<Product>(), Arg.Any<CancellationToken>())
            .Returns(callInfo => callInfo.Arg<Product>());

        // Act
        await _sut.Handle(Command, CancellationToken.None);

        // Assert
        await _productRepository.Received(1).AddAsync(
            Arg.Is<Product>(p =>
                p.Name == Command.Name &&
                p.Price == Command.Price &&
                p.CategoryId == Command.CategoryId),
            Arg.Any<CancellationToken>());
    }
}
```

### More complex handler with multiple steps and UnitOfWork

When a handler has multiple sequential validations (fetch user → fetch apartment → check overlap → save), test each step's failure path independently:

```csharp
public class ReserveBookingCommandHandlerTests
{
    private static readonly DateTime UtcNow = DateTime.UtcNow;

    // Shared command used across all tests
    private static readonly ReserveBookingCommand Command = new(
        UserId: Guid.NewGuid(),
        ApartmentId: Guid.NewGuid(),
        StartDate: new DateOnly(2024, 1, 1),
        EndDate: new DateOnly(2024, 1, 10));

    private readonly IUserRepository _userRepository;
    private readonly IApartmentRepository _apartmentRepository;
    private readonly IBookingRepository _bookingRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IDateTimeProvider _dateTimeProvider;
    private readonly ReserveBookingCommandHandler _sut;

    public ReserveBookingCommandHandlerTests()
    {
        _userRepository = Substitute.For<IUserRepository>();
        _apartmentRepository = Substitute.For<IApartmentRepository>();
        _bookingRepository = Substitute.For<IBookingRepository>();
        _unitOfWork = Substitute.For<IUnitOfWork>();
        _dateTimeProvider = Substitute.For<IDateTimeProvider>();
        _dateTimeProvider.UtcNow.Returns(UtcNow);

        _sut = new ReserveBookingCommandHandler(
            _userRepository,
            _apartmentRepository,
            _bookingRepository,
            _unitOfWork,
            new PricingService(),
            _dateTimeProvider);
    }

    // Step 1: User must exist
    [Fact]
    public async Task Handle_WhenUserNotFound_ReturnsFailure()
    {
        // Arrange — only set up what's needed for step 1 to fail
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns((User?)null);

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be(UserErrors.NotFound);
    }

    // Step 2: Apartment must exist (step 1 passes)
    [Fact]
    public async Task Handle_WhenApartmentNotFound_ReturnsFailure()
    {
        // Arrange — step 1 passes, step 2 fails
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns(UserData.Create());

        _apartmentRepository
            .GetByIdAsync(Command.ApartmentId, Arg.Any<CancellationToken>())
            .Returns((Apartment?)null);

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be(ApartmentErrors.NotFound);
    }

    // Step 3: No overlap (steps 1 and 2 pass)
    [Fact]
    public async Task Handle_WhenApartmentIsAlreadyBooked_ReturnsFailure()
    {
        // Arrange — steps 1 and 2 pass, step 3 fails
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns(UserData.Create());

        _apartmentRepository
            .GetByIdAsync(Command.ApartmentId, Arg.Any<CancellationToken>())
            .Returns(ApartmentData.Create());

        _bookingRepository
            .IsOverlappingAsync(Arg.Any<Apartment>(), Arg.Any<DateRange>(), Arg.Any<CancellationToken>())
            .Returns(true);

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be(BookingErrors.Overlap);
    }

    // Step 4: Save must succeed (all prior steps pass)
    [Fact]
    public async Task Handle_WhenSaveThrows_ReturnsFailure()
    {
        // Arrange — all steps pass, save fails
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns(UserData.Create());

        _apartmentRepository
            .GetByIdAsync(Command.ApartmentId, Arg.Any<CancellationToken>())
            .Returns(ApartmentData.Create());

        _bookingRepository
            .IsOverlappingAsync(Arg.Any<Apartment>(), Arg.Any<DateRange>(), Arg.Any<CancellationToken>())
            .Returns(false);

        _unitOfWork
            .SaveChangesAsync(Arg.Any<CancellationToken>())
            .ThrowsAsync(new ConcurrencyException("Concurrency", new Exception()));

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
    }

    // Happy path: everything succeeds → success result
    [Fact]
    public async Task Handle_WhenAllStepsPass_ReturnsSuccessWithBookingId()
    {
        // Arrange — all steps pass
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns(UserData.Create());

        _apartmentRepository
            .GetByIdAsync(Command.ApartmentId, Arg.Any<CancellationToken>())
            .Returns(ApartmentData.Create());

        _bookingRepository
            .IsOverlappingAsync(Arg.Any<Apartment>(), Arg.Any<DateRange>(), Arg.Any<CancellationToken>())
            .Returns(false);

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBe(Guid.Empty);
    }

    // Verify interaction: booking was saved
    [Fact]
    public async Task Handle_WhenAllStepsPass_CallsBookingRepositoryAdd()
    {
        // Arrange — same as happy path
        _userRepository
            .GetByIdAsync(Command.UserId, Arg.Any<CancellationToken>())
            .Returns(UserData.Create());

        _apartmentRepository
            .GetByIdAsync(Command.ApartmentId, Arg.Any<CancellationToken>())
            .Returns(ApartmentData.Create());

        _bookingRepository
            .IsOverlappingAsync(Arg.Any<Apartment>(), Arg.Any<DateRange>(), Arg.Any<CancellationToken>())
            .Returns(false);

        // Act
        var result = await _sut.Handle(Command, CancellationToken.None);

        // Assert
        _bookingRepository.Received(1).Add(
            Arg.Is<Booking>(b => b.Id == result.Value));
    }
}
```

---

## Query Handler Tests

Queries are typically simpler — fewer steps and no side effects to verify. Still use the same step-by-step approach:

```csharp
public class GetProductByIdQueryHandlerTests
{
    private static readonly Guid ProductId = Guid.NewGuid();
    private static readonly GetProductByIdQuery Query = new(ProductId);

    private readonly IProductRepository _repository;
    private readonly GetProductByIdQueryHandler _sut;

    public GetProductByIdQueryHandlerTests()
    {
        _repository = Substitute.For<IProductRepository>();
        _sut = new GetProductByIdQueryHandler(_repository);
    }

    [Fact]
    public async Task Handle_WhenProductNotFound_ThrowsNotFoundException()
    {
        // Arrange
        _repository
            .GetByIdAsync(ProductId, Arg.Any<CancellationToken>())
            .Returns((Product?)null);

        // Act
        var act = async () => await _sut.Handle(Query, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<NotFoundException>();
    }

    [Fact]
    public async Task Handle_WhenProductExists_ReturnsProductResponse()
    {
        // Arrange
        var product = ProductData.Create(id: ProductId, name: "Widget", price: 9.99m);
        _repository
            .GetByIdAsync(ProductId, Arg.Any<CancellationToken>())
            .Returns(product);

        // Act
        var result = await _sut.Handle(Query, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(ProductId);
        result.Name.Should().Be("Widget");
    }
}
```

---

## FluentValidation Validator Tests

```csharp
using FluentValidation.TestHelper;

public class CreateProductCommandValidatorTests
{
    private readonly CreateProductCommandValidator _sut = new();

    [Fact]
    public void Validate_WithValidCommand_PassesValidation()
    {
        var command = new CreateProductCommand("Widget", "Description", 9.99m, Guid.NewGuid());

        var result = _sut.TestValidate(command);

        result.ShouldNotHaveAnyValidationErrors();
    }

    [Fact]
    public void Validate_WithEmptyName_HasValidationError()
    {
        var command = new CreateProductCommand("", "Description", 9.99m, Guid.NewGuid());

        var result = _sut.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.Name);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100.50)]
    public void Validate_WithInvalidPrice_HasValidationError(decimal price)
    {
        var command = new CreateProductCommand("Widget", "Description", price, Guid.NewGuid());

        var result = _sut.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.Price);
    }

    [Fact]
    public void Validate_WithEmptyGuid_HasValidationError()
    {
        var command = new CreateProductCommand("Widget", "Description", 9.99m, Guid.Empty);

        var result = _sut.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.CategoryId);
    }
}
```

---

## Adapting Templates to Real Handlers

1. **Read the handler class** — trace through the `Handle` method step by step. List every guard clause, repository call, and domain operation.
2. **Write one test per step** — one for each failure point + one for success + one for each important repository interaction.
3. **Identify all dependencies** — create NSubstitute mocks for each interface in the constructor.
4. **Create test data factories** in `TestData/` for each entity the handler uses.
5. **Define a static `Command`** at class level using the actual command record's fields.
6. **Use `IDateTimeProvider`** — if the handler uses DateTime, always inject it as an interface and mock with `_dateTimeProvider.UtcNow.Returns(UtcNow)`.
7. **Use `IUnitOfWork`** — if the project uses Unit of Work pattern, mock `SaveChangesAsync` and test what happens when it throws.
