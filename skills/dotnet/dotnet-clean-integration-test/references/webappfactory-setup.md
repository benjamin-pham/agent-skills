# WebApplicationFactory, IntegrationTestBase, and JWT Helper

## CustomWebApplicationFactory.cs

Place in `tests/{ProjectName}.IntegrationTests/`:

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Testcontainers.PostgreSql;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.IntegrationTests;

public class CustomWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("testuser")
        .WithPassword("testpass")
        .Build();

    public string ConnectionString => _dbContainer.GetConnectionString();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Remove existing DbContext registration
            var dbContextDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (dbContextDescriptor != null)
                services.Remove(dbContextDescriptor);

            // Register test database connection
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(ConnectionString));
        });
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        // Apply all EF Core migrations to create the schema
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

**SQL Server variant:** Replace `Testcontainers.PostgreSql` with `Testcontainers.MsSql`, `PostgreSqlContainer` with `MsSqlContainer`, `PostgreSqlBuilder` with `MsSqlBuilder`, and `UseNpgsql` with `UseSqlServer`.

---

## IntegrationTestBase.cs

Place in `tests/{ProjectName}.IntegrationTests/`:

```csharp
using Npgsql;
using Respawn;
using {ProjectName}.Infrastructure.Data;

namespace {ProjectName}.IntegrationTests;

public abstract class IntegrationTestBase : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    protected readonly HttpClient Client;
    protected readonly CustomWebApplicationFactory Factory;
    private Respawner _respawner = default!;

    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient(new() { AllowAutoRedirect = false });
    }

    public async Task InitializeAsync()
    {
        // Reset DB state before each test via Respawn
        await using var connection = new NpgsqlConnection(Factory.ConnectionString);
        await connection.OpenAsync();

        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            // Exclude migration history table from reset
            TablesToIgnore = ["__EFMigrationsHistory"]
        });

        await _respawner.ResetAsync(connection);
    }

    public Task DisposeAsync()
    {
        Client.Dispose();
        return Task.CompletedTask;
    }

    // ── Seed helpers ──────────────────────────────────────────────────────────

    /// <summary>Seeds a single entity. The entity's Id will be set after this call.</summary>
    protected async Task SeedAsync<T>(T entity) where T : class
    {
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Set<T>().Add(entity);
        await db.SaveChangesAsync();
    }

    /// <summary>Seeds multiple entities.</summary>
    protected async Task SeedAsync<T>(IEnumerable<T> entities) where T : class
    {
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Set<T>().AddRange(entities);
        await db.SaveChangesAsync();
    }

    // ── DB query helper ───────────────────────────────────────────────────────

    /// <summary>Queries the DB directly — useful for verifying persistence after a POST/PUT/DELETE.</summary>
    protected async Task<T> GetFromDbAsync<T>(Func<AppDbContext, Task<T>> query)
    {
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await query(db);
    }

    // ── Authentication ────────────────────────────────────────────────────────

    /// <summary>Attaches a valid JWT to all subsequent requests from this test.</summary>
    protected void AuthenticateAs(string userId, string role = "User")
    {
        using var scope = Factory.Services.CreateScope();
        var config = scope.ServiceProvider.GetRequiredService<IConfiguration>();
        var token = JwtTokenHelper.GenerateToken(userId, role, config);
        Client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
    }

    /// <summary>Removes the JWT header — useful when testing unauthenticated access mid-test.</summary>
    protected void Unauthenticate()
    {
        Client.DefaultRequestHeaders.Authorization = null;
    }

    // ── Repository helper ─────────────────────────────────────────────────────

    /// <summary>Resolves a service from the DI container — useful for repository tests.</summary>
    protected T GetService<T>() where T : notnull
    {
        return Factory.Services.CreateScope().ServiceProvider.GetRequiredService<T>();
    }
}
```

---

## Helpers/JwtTokenHelper.cs

Place in `tests/{ProjectName}.IntegrationTests/Helpers/`:

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;

namespace {ProjectName}.IntegrationTests.Helpers;

public static class JwtTokenHelper
{
    public static string GenerateToken(string userId, string role, IConfiguration config)
    {
        var jwtSettings = config.GetSection("JwtSettings");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSettings["Key"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId),
            new Claim(ClaimTypes.Role, role),
            new Claim(JwtRegisteredClaimNames.Sub, userId),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };

        var token = new JwtSecurityToken(
            issuer: jwtSettings["Issuer"],
            audience: jwtSettings["Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

This generates tokens signed with the **same key** as the running application reads from `appsettings.json` (or `appsettings.Testing.json`). Because `WebApplicationFactory` runs the real app configuration pipeline, the keys will always match.

---

## How These Three Pieces Fit Together

```
xUnit test runner
  └── Instantiates CustomWebApplicationFactory once per test class
        └── Starts PostgreSQL container
        └── Runs EF Core migrations
        └── For each test:
              └── IntegrationTestBase.InitializeAsync() → Respawn resets DB
              └── Test runs → uses Client (HTTP) or GetService<T> (DI)
              └── IntegrationTestBase.DisposeAsync() → disposes HttpClient
  └── CustomWebApplicationFactory.DisposeAsync() → stops container
```

`IClassFixture<CustomWebApplicationFactory>` means xUnit shares **one** factory (and one Docker container) across all test methods in a class. This keeps test runs fast — container startup happens once.

Respawn runs before each test, wiping all data from `public` schema tables. This isolates tests from each other without the overhead of recreating the database schema each time.
