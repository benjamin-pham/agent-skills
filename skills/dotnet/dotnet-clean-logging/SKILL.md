---
name: dotnet-clean-logging
description: >
  Add structured logging statements to ASP.NET Core services, controllers, and
  handlers using ILogger<T> with Serilog backend. Covers DI injection, choosing
  the right log level, message templates with {Property} and {@Property}
  (destructure object to JSON), and exception logging patterns. Trigger whenever
  the user wants to add log lines, log a request/response, add logging to a
  service or handler, debug with logs — including Vietnamese like "thêm logging",
  "ghi log cho service", "log request response", "thêm log vào controller".
---

# Structured Logging with ILogger\<T>

This skill adds logging statements to existing code. The project uses **Serilog** as the backend, so all logs are structured JSON. ILogger<T> is the facade — Serilog captures the structured properties behind the scenes.

## Inject ILogger\<T>

Use constructor injection. The `T` matches the class name so each log line automatically includes `SourceContext`:

```csharp
public class OrderService(
    IOrderRepository orderRepository,
    ILogger<OrderService> logger)
{
    // logger is ready to use
}
```

For minimal APIs or static contexts where DI isn't available, resolve from `IServiceProvider`:

```csharp
app.MapGet("/health", (ILogger<Program> logger) =>
{
    logger.LogInformation("Health check requested");
    return Results.Ok();
});
```

## Log Levels

Pick the level that matches the **audience and urgency**:

| Level | When to use | Example |
|-------|------------|---------|
| `LogTrace` | Very low-level diagnostic detail — usually off in production | Entering method `CalculateDiscount` |
| `LogDebug` | Internal state useful during development | Loaded {Count} items from cache |
| `LogInformation` | Normal business events worth recording | Order {OrderId} created by {UserId} |
| `LogWarning` | Something unexpected but recoverable | Retry attempt {Attempt} for payment {PaymentId} |
| `LogError` | Operation failed, needs attention | Failed to process order {OrderId} |
| `LogCritical` | Application is about to crash or data loss | Database connection pool exhausted |

Rules of thumb:
- **Information** is the default for business events (order created, user logged in, email sent)
- **Warning** means "look at this soon" — retries, fallback paths, deprecated usage
- **Error** means "this operation failed" — always include the exception when there is one
- **Debug/Trace** are for development only — keep them out of hot paths in production

## Message Templates — The Core Concept

Serilog uses **message templates**, not string formatting. This is the single most important thing to get right.

### Do: use named placeholders

```csharp
logger.LogInformation("Order {OrderId} created for {Amount:C}", order.Id, order.Amount);
```

Serilog captures `OrderId` and `Amount` as **separate structured properties** in the JSON output. This makes them searchable and filterable in log aggregation tools (Seq, Elasticsearch, Grafana Loki).

### Don't: use string interpolation

```csharp
// BAD — Serilog sees one flat string, no structured properties
logger.LogInformation($"Order {order.Id} created for {order.Amount}");
```

String interpolation bakes values into the message before Serilog sees them, destroying the structured data. The log still appears but you lose the ability to filter by `OrderId` or `Amount`.

### Scalar vs Object: `{Property}` vs `{@Property}`

This is Serilog-specific and critical:

| Syntax | Behavior | Use when |
|--------|----------|----------|
| `{Property}` | Calls `.ToString()` on the value | Scalars: int, string, Guid, enum, DateTime |
| `{@Property}` | **Destructures** the object into JSON properties | Complex objects: DTOs, request/response bodies, entities |

```csharp
// Scalar — just the ID value
logger.LogInformation("Processing order {OrderId}", orderId);

// Destructure — the entire object becomes JSON in the log
logger.LogInformation("Received request {@Request}", createOrderRequest);
// Output: { "Request": { "CustomerId": 42, "Items": [...], "Total": 99.9 } }

// Mixed — scalar ID + destructured payload
logger.LogWarning("Validation failed for order {OrderId} with payload {@Payload}",
    orderId, createOrderRequest);
```

When to use `{@Property}`:
- Logging a DTO or request/response body
- Logging an object for debugging (but be mindful of size and sensitive data)
- Logging a collection of items

When NOT to use `{@Property}`:
- Simple scalars (int, string, Guid) — `{Property}` is cleaner
- Entities with navigation properties — EF Core lazy loading can trigger unexpected queries
- Objects containing sensitive data (passwords, tokens) — filter or redact first

### Format specifiers

Serilog supports standard .NET format strings after a colon:

```csharp
logger.LogInformation("Payment of {Amount:C} received at {Timestamp:yyyy-MM-dd}", amount, DateTime.UtcNow);
// Output: Payment of $99.90 received at 2026-03-21
```

## Exception Logging

Always pass the exception as the **first argument** — Serilog captures the full stack trace as a structured property:

```csharp
try
{
    await orderRepository.CreateAsync(order, cancellationToken);
    logger.LogInformation("Order {OrderId} saved successfully", order.Id);
}
catch (DbUpdateException ex)
{
    logger.LogError(ex, "Failed to save order {OrderId}", order.Id);
    throw; // re-throw after logging — don't swallow
}
```

The `ex` goes in the first parameter, NOT in the message template. Serilog stores the exception separately with full stack trace, message, and inner exceptions.

### Pattern for service methods

```csharp
public async Task<OrderResponse> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
{
    logger.LogInformation("Creating order for customer {CustomerId} with {ItemCount} items",
        request.CustomerId, request.Items.Count);

    try
    {
        var order = MapToEntity(request);
        await orderRepository.CreateAsync(order, ct);

        logger.LogInformation("Order {OrderId} created successfully", order.Id);
        return MapToResponse(order);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to create order for customer {CustomerId}", request.CustomerId);
        throw;
    }
}
```

## Where to Add Logs

Follow this guide based on what the user asks:

| Layer | What to log | Level |
|-------|------------|-------|
| **Controller / Endpoint** | Request received, response returned (if not using `UseSerilogRequestLogging`) | Information |
| **Application Service** | Business operation start/end, validation failures | Information / Warning |
| **Domain** | Rarely — domain should be pure. Only if domain events exist | Debug |
| **Infrastructure** | External calls (API, DB, cache), retries, fallbacks | Information / Warning / Error |
| **Middleware** | Cross-cutting: auth failures, rate limiting, unhandled exceptions | Warning / Error |

## Important Reminders

- The project MUST already have Serilog configured. This skill only adds log statements to existing code.
- **Never log sensitive data**: passwords, tokens, credit card numbers, PII. If the DTO contains sensitive fields, log only safe properties or create a sanitized projection.
- **Be mindful of `{@Property}` on large objects** — destructuring a 1000-item list will bloat your logs. Log the count instead: `{ItemCount}` items.
- **Property naming**: use PascalCase for property names in templates (`{OrderId}`, not `{orderId}`) — this matches C# conventions and keeps log properties consistent.
- **Don't log inside tight loops** — if you need to, use `LogDebug` or `LogTrace` so it can be filtered out in production.
