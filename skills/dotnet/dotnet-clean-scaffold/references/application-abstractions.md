# Application Layer — Abstractions & DI

Create all files below. Replace `{ProjectName}` with the actual project name.

---

## src/{ProjectName}.Application/Abstractions/Messaging/ICommand.cs

```csharp
using MediatR;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Application.Abstractions.Messaging;

public interface ICommand : IRequest<Result>
{
}

public interface ICommand<TResponse> : IRequest<Result<TResponse>>
{
}
```

---

## src/{ProjectName}.Application/Abstractions/Messaging/IQuery.cs

```csharp
using MediatR;
using {ProjectName}.Domain.Abstractions;

namespace {ProjectName}.Application.Abstractions.Messaging;

public interface IQuery<TResponse> : IRequest<Result<TResponse>>
{
}
```

---

## src/{ProjectName}.Application/Abstractions/Messaging/ICommandHandler.cs

```csharp
using MediatR;

namespace {ProjectName}.Application.Abstractions.Messaging;

public interface ICommandHandler<TCommand> : IRequestHandler<TCommand, Result>
    where TCommand : ICommand
{
}

public interface ICommandHandler<TCommand, TResponse> : IRequestHandler<TCommand, Result<TResponse>>
    where TCommand : ICommand<TResponse>
{
}
```

---

## src/{ProjectName}.Application/Abstractions/Messaging/IQueryHandler.cs

```csharp
using MediatR;

namespace {ProjectName}.Application.Abstractions.Messaging;

public interface IQueryHandler<TQuery, TResponse> : IRequestHandler<TQuery, Result<TResponse>>
    where TQuery : IQuery<TResponse>
{
}
```

---

## src/{ProjectName}.Application/Abstractions/Data/ISqlConnectionFactory.cs

```csharp
using System.Data;

namespace {ProjectName}.Application.Abstractions.Data;

public interface ISqlConnectionFactory
{
    IDbConnection CreateConnection();
}
```

---

## src/{ProjectName}.Application/Behaviors/ValidationBehavior.cs

This pipeline behavior runs FluentValidation validators before every MediatR request.
It throws a `ValidationException` if any validator fails — catch it in global middleware.

```csharp
using FluentValidation;
using MediatR;

namespace {ProjectName}.Application.Behaviors;

internal sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next(cancellationToken);

        var context = new ValidationContext<TRequest>(request);

        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next(cancellationToken);
    }
}
```

---

## src/{ProjectName}.Application/DependencyInjection.cs

```csharp
using System.Reflection;
using FluentValidation;
using Microsoft.Extensions.DependencyInjection;
using {ProjectName}.Application.Behaviors;

namespace {ProjectName}.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            config.AddOpenBehavior(typeof(ValidationBehavior<,>));
        });

        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        return services;
    }
}
```
