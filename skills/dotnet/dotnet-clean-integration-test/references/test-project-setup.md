# Integration Test Project Setup

## Directory Structure

```
{SolutionRoot}/
├── src/
│   ├── {ProjectName}.Domain/
│   ├── {ProjectName}.Application/
│   ├── {ProjectName}.Infrastructure/
│   └── {ProjectName}.API/
├── tests/
│   └── {ProjectName}.IntegrationTests/
│       ├── {ProjectName}.IntegrationTests.csproj
│       ├── GlobalUsings.cs
│       ├── CustomWebApplicationFactory.cs
│       ├── IntegrationTestBase.cs
│       ├── Helpers/
│       │   └── JwtTokenHelper.cs
│       └── Features/
│           └── {Feature}/
│               ├── {Feature}EndpointTests.cs
│               └── {Feature}RepositoryTests.cs
├── Directory.Build.props
└── Directory.Packages.props
```

## Step 1 — Add Test Packages to Directory.Packages.props

If a `Label="Testing"` group already exists (from unit tests), extend it. Otherwise add a new group:

```xml
<ItemGroup Label="IntegrationTesting">
  <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
  <PackageVersion Include="xunit" Version="2.9.3" />
  <PackageVersion Include="xunit.runner.visualstudio" Version="3.0.1" />
  <PackageVersion Include="FluentAssertions" Version="8.0.1" />
  <PackageVersion Include="coverlet.collector" Version="6.0.4" />
  <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.0.0" />
  <PackageVersion Include="Testcontainers.PostgreSql" Version="3.10.0" />
  <PackageVersion Include="Npgsql" Version="9.0.0" />
  <PackageVersion Include="Respawn" Version="6.2.1" />
</ItemGroup>
```

**SQL Server variant:** Replace `Testcontainers.PostgreSql` with `Testcontainers.MsSql` and `Npgsql` with `Microsoft.Data.SqlClient`.

Check NuGet for latest stable versions — these go stale quickly.

> If unit test packages are already in `Directory.Packages.props`, skip duplicate entries (`xunit`, `FluentAssertions`, etc.).

## Step 2 — Create the IntegrationTests Project

```bash
dotnet new xunit -n {ProjectName}.IntegrationTests -o tests/{ProjectName}.IntegrationTests
dotnet sln add tests/{ProjectName}.IntegrationTests
dotnet add tests/{ProjectName}.IntegrationTests reference src/{ProjectName}.API
```

The IntegrationTests project references the API project to access `Program` and the entire dependency graph transitively.

## Step 3 — Update the .csproj File

After `dotnet new xunit`, the generated .csproj has explicit package versions. Replace it with Central Package Management style:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="coverlet.collector" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
    <PackageReference Include="Testcontainers.PostgreSql" />
    <PackageReference Include="Npgsql" />
    <PackageReference Include="Respawn" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.API\{ProjectName}.API.csproj" />
  </ItemGroup>

</Project>
```

## Step 4 — Expose Program Class for WebApplicationFactory

`WebApplicationFactory<Program>` needs access to the `Program` class. Add this **at the very end** of the API project's `Program.cs` (after all `app.Run()` calls):

```csharp
// Expose Program for integration tests
public partial class Program { }
```

This is a common pattern — it makes the `Program` type accessible without changing the `internal` visibility of the entire class.

## Step 5 — Clean Up Generated Files

Delete the generated `UnitTest1.cs`. Then create `GlobalUsings.cs`:

```csharp
global using System.Net;
global using System.Net.Http.Headers;
global using System.Net.Http.Json;
global using Xunit;
global using FluentAssertions;
global using Microsoft.Extensions.DependencyInjection;
global using {ProjectName}.Domain.Entities;
global using {ProjectName}.Infrastructure.Data;
```

## Step 6 — Verify Setup

```bash
dotnet build
dotnet test tests/{ProjectName}.IntegrationTests --no-build
```

With no test classes yet, this should succeed with "No tests found". The first real test run will pull the PostgreSQL Docker image — expect 30–60 seconds on first run.
