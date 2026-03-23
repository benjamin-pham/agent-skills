# Endpoint Infrastructure (One-Time Setup)

## IEndpoint Interface

`src/{ProjectName}.API/Endpoints/IEndpoint.cs`

```csharp
namespace {ProjectName}.API.Endpoints;

public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder app);
}
```

## EndpointExtensions — Auto-Registration

`src/{ProjectName}.API/Endpoints/EndpointExtensions.cs`

```csharp
using System.Reflection;

namespace {ProjectName}.API.Endpoints;

public static class EndpointExtensions
{
    public static IEndpointRouteBuilder MapEndpoints(this IEndpointRouteBuilder app)
    {
        var endpointTypes = Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => t is { IsAbstract: false, IsInterface: false }
                        && typeof(IEndpoint).IsAssignableFrom(t));

        foreach (var type in endpointTypes)
        {
            var endpoint = (IEndpoint)Activator.CreateInstance(type)!;
            endpoint.MapEndpoint(app);
        }

        return app;
    }
}
```

This scans the API assembly at startup, finds every concrete class implementing `IEndpoint`, creates an instance, and calls `MapEndpoint`. No manual registration needed — just create a new class and it's automatically discovered.

## Program.cs Update

Replace the manual endpoint registration block:

```csharp
// BEFORE (old pattern — remove these lines):
// app.MapProductEndpoints();
// app.MapCategoryEndpoints();

// AFTER (new pattern — single line):
app.MapEndpoints();
```

Add the using directive at the top of `Program.cs`:

```csharp
using {ProjectName}.API.Endpoints;
```

The full `Program.cs` endpoint section becomes:

```csharp
// Map endpoints

app.MapEndpoints();

app.Run();
```
