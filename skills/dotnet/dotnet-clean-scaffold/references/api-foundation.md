# API Layer — Foundation

Create all files below. Replace `{ProjectName}` with the actual project name.

---

## src/{ProjectName}.API/Endpoints/IEndpoint.cs

```csharp
namespace {ProjectName}.API.Endpoints;

public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder app);
}
```

---

## src/{ProjectName}.API/Endpoints/EndpointExtensions.cs

Scans the assembly for all `IEndpoint` implementations and registers them automatically.
This means `Program.cs` never needs to reference individual endpoint classes.

```csharp
using System.Reflection;

namespace {ProjectName}.API.Endpoints;

public static class EndpointExtensions
{
    public static IServiceCollection AddEndpoints(
        this IServiceCollection services,
        Assembly assembly)
    {
        var endpointTypes = assembly
            .GetTypes()
            .Where(t => t is { IsAbstract: false, IsInterface: false }
                        && t.IsAssignableTo(typeof(IEndpoint)));

        foreach (var type in endpointTypes)
            services.AddTransient(typeof(IEndpoint), type);

        return services;
    }

    public static IApplicationBuilder MapEndpoints(
        this WebApplication app,
        RouteGroupBuilder? routeGroupBuilder = null)
    {
        var endpoints = app.Services.GetRequiredService<IEnumerable<IEndpoint>>();

        IEndpointRouteBuilder builder = routeGroupBuilder is null ? app : routeGroupBuilder;

        foreach (var endpoint in endpoints)
            endpoint.MapEndpoint(builder);

        return app;
    }
}
```

---

## src/{ProjectName}.API/Extensions/ExceptionHandlingMiddleware.cs

Catches unhandled exceptions (including FluentValidation's `ValidationException`)
and returns a structured problem response.

```csharp
using FluentValidation;
using Microsoft.AspNetCore.Mvc;

namespace {ProjectName}.API.Extensions;

internal sealed class ExceptionHandlingMiddleware(
    RequestDelegate next,
    ILogger<ExceptionHandlingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (ValidationException ex)
        {
            logger.LogWarning(ex, "Validation error on {Path}", context.Request.Path);

            var problemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Type = "ValidationFailure",
                Title = "Validation error",
                Detail = "One or more validation errors occurred"
            };

            problemDetails.Extensions["errors"] = ex.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray());

            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(problemDetails);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception on {Path}", context.Request.Path);

            var problemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Server error"
            };

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(problemDetails);
        }
    }
}
```

---

## src/{ProjectName}.API/Program.cs

```csharp
using System.Reflection;
using Scalar.AspNetCore;
using Serilog;
using {ProjectName}.API.Endpoints;
using {ProjectName}.API.Extensions;
using {ProjectName}.Application;
using {ProjectName}.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// ── Serilog ──────────────────────────────────────────────────────────────────
builder.Host.UseSerilog((context, loggerConfig) =>
    loggerConfig.ReadFrom.Configuration(context.Configuration));

// ── Services ─────────────────────────────────────────────────────────────────
builder.Services.AddOpenApi();
builder.Services.AddEndpoints(Assembly.GetExecutingAssembly());
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

// ── Pipeline ─────────────────────────────────────────────────────────────────
var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
}

app.UseSerilogRequestLogging();
app.UseHttpsRedirection();
app.MapEndpoints();

app.Run();
```

---

## src/{ProjectName}.API/appsettings.json

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName"]
  },
  "AllowedHosts": "*"
}
```

---

## src/{ProjectName}.API/appsettings.Development.json

```json
{
  "ConnectionStrings": {
    "Database": "Host=localhost;Database={projectname_lowercase};Username=postgres;Password=your_password"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft.EntityFrameworkCore.Database.Command": "Information"
      }
    }
  }
}
```

Replace `{projectname_lowercase}` with the actual database name in lowercase.
Never commit real credentials — add `appsettings.Development.json` to `.gitignore`.

---

## .gitignore (solution root)

Create a `.gitignore` at the solution root:

```bash
dotnet new gitignore
```

Then add these lines to it:

```
appsettings.Development.json
appsettings.*.json
!appsettings.json
logs/
```
