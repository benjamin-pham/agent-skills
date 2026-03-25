# Test Projects Scaffold

Five test projects live under `tests/`, each with a distinct scope:

| Project | Scope | Key packages |
|---------|-------|--------------|
| `{ProjectName}.Domain.UnitTests` | Pure domain logic — entities, value objects, Result | xunit, FluentAssertions |
| `{ProjectName}.Application.UnitTests` | Handlers, validators, behaviors — with mocked interfaces | xunit, FluentAssertions, NSubstitute |
| `{ProjectName}.Infrastructure.IntegrationTests` | EF Core mappings, repositories — real DB via Testcontainers | xunit, FluentAssertions, Testcontainers |
| `{ProjectName}.API.IntegrationTests` | Full HTTP stack via WebApplicationFactory | xunit, FluentAssertions, Mvc.Testing, Testcontainers |
| `{ProjectName}.ArchitectureTests` | Layer dependency rules enforced statically | xunit, FluentAssertions, NetArchTest.Rules |

---

## Step 1 — Create the five projects

Run from the solution root (where the `.slnx` file lives):

```bash
dotnet new xunit -n {ProjectName}.Domain.UnitTests            -o tests/{ProjectName}.Domain.UnitTests            --framework net10.0
dotnet new xunit -n {ProjectName}.Application.UnitTests       -o tests/{ProjectName}.Application.UnitTests       --framework net10.0
dotnet new xunit -n {ProjectName}.Infrastructure.IntegrationTests -o tests/{ProjectName}.Infrastructure.IntegrationTests --framework net10.0
dotnet new xunit -n {ProjectName}.API.IntegrationTests        -o tests/{ProjectName}.API.IntegrationTests        --framework net10.0
dotnet new xunit -n {ProjectName}.ArchitectureTests           -o tests/{ProjectName}.ArchitectureTests           --framework net10.0
```

---

## Step 2 — Update the .slnx file

The `.slnx` format does not support `dotnet sln add`. Edit the file directly and add the test projects inside a `<Folder Name="tests">` element:

```xml
<Solution>
  <Folder Name="src">
    <Project Path="src/{ProjectName}.Domain/{ProjectName}.Domain.csproj" />
    <Project Path="src/{ProjectName}.Application/{ProjectName}.Application.csproj" />
    <Project Path="src/{ProjectName}.Infrastructure/{ProjectName}.Infrastructure.csproj" />
    <Project Path="src/{ProjectName}.API/{ProjectName}.API.csproj" />
  </Folder>
  <Folder Name="tests">
    <Project Path="tests/{ProjectName}.Domain.UnitTests/{ProjectName}.Domain.UnitTests.csproj" />
    <Project Path="tests/{ProjectName}.Application.UnitTests/{ProjectName}.Application.UnitTests.csproj" />
    <Project Path="tests/{ProjectName}.Infrastructure.IntegrationTests/{ProjectName}.Infrastructure.IntegrationTests.csproj" />
    <Project Path="tests/{ProjectName}.API.IntegrationTests/{ProjectName}.API.IntegrationTests.csproj" />
    <Project Path="tests/{ProjectName}.ArchitectureTests/{ProjectName}.ArchitectureTests.csproj" />
  </Folder>
</Solution>
```

---

## Step 3 — Add project references

```bash
dotnet add tests/{ProjectName}.Domain.UnitTests               reference src/{ProjectName}.Domain
dotnet add tests/{ProjectName}.Application.UnitTests          reference src/{ProjectName}.Application
dotnet add tests/{ProjectName}.Infrastructure.IntegrationTests reference src/{ProjectName}.Infrastructure
dotnet add tests/{ProjectName}.API.IntegrationTests           reference src/{ProjectName}.API
dotnet add tests/{ProjectName}.ArchitectureTests              reference src/{ProjectName}.Domain
dotnet add tests/{ProjectName}.ArchitectureTests              reference src/{ProjectName}.Application
dotnet add tests/{ProjectName}.ArchitectureTests              reference src/{ProjectName}.Infrastructure
dotnet add tests/{ProjectName}.ArchitectureTests              reference src/{ProjectName}.API
```

`ArchitectureTests` references all four source projects so `NetArchTest` can inspect every assembly.

---

## Step 4 — Replace generated .csproj files (Central Package Management)

`dotnet new xunit` generates `.csproj` files with explicit `Version` attributes. Replace each with the CPM-style template below (no `<TargetFramework>`, `<Nullable>`, or `<ImplicitUsings>` — all inherited from `Directory.Build.props`).

### {ProjectName}.Domain.UnitTests.csproj

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
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Domain\{ProjectName}.Domain.csproj" />
  </ItemGroup>

</Project>
```

Domain unit tests do **not** need NSubstitute — entities and value objects are tested directly.

### {ProjectName}.Application.UnitTests.csproj

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
    <PackageReference Include="NSubstitute" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Application\{ProjectName}.Application.csproj" />
  </ItemGroup>

</Project>
```

### {ProjectName}.Infrastructure.IntegrationTests.csproj

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
    <PackageReference Include="Testcontainers.PostgreSql" />
    <PackageReference Include="Npgsql" />
    <PackageReference Include="Respawn" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Infrastructure\{ProjectName}.Infrastructure.csproj" />
  </ItemGroup>

</Project>
```

**SQL Server variant:** Replace `Testcontainers.PostgreSql` + `Npgsql` with `Testcontainers.MsSql` + `Microsoft.Data.SqlClient`. Update `Directory.Packages.props` accordingly.

### {ProjectName}.API.IntegrationTests.csproj

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
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
    <PackageReference Include="Testcontainers.PostgreSql" />
    <PackageReference Include="Npgsql" />
    <PackageReference Include="Respawn" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.API\{ProjectName}.API.csproj" />
  </ItemGroup>

</Project>
```

### {ProjectName}.ArchitectureTests.csproj

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
    <PackageReference Include="NetArchTest.Rules" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Domain\{ProjectName}.Domain.csproj" />
    <ProjectReference Include="..\..\src\{ProjectName}.Application\{ProjectName}.Application.csproj" />
    <ProjectReference Include="..\..\src\{ProjectName}.Infrastructure\{ProjectName}.Infrastructure.csproj" />
    <ProjectReference Include="..\..\src\{ProjectName}.API\{ProjectName}.API.csproj" />
  </ItemGroup>

</Project>
```

---

## Step 5 — Expose Program for WebApplicationFactory

`{ProjectName}.API.IntegrationTests` needs access to the `Program` class. Add this at the **very end** of `src/{ProjectName}.API/Program.cs` (after `app.Run()`):

```csharp
// Expose Program for integration tests
public partial class Program { }
```

---

## Step 6 — Delete generated boilerplate and create GlobalUsings.cs

`dotnet new xunit` generates `UnitTest1.cs` and `GlobalUsings.cs` in every project. Delete the test file and replace `GlobalUsings.cs` with project-appropriate usings.

```bash
rm tests/{ProjectName}.Domain.UnitTests/UnitTest1.cs
rm tests/{ProjectName}.Application.UnitTests/UnitTest1.cs
rm tests/{ProjectName}.Infrastructure.IntegrationTests/UnitTest1.cs
rm tests/{ProjectName}.API.IntegrationTests/UnitTest1.cs
rm tests/{ProjectName}.ArchitectureTests/UnitTest1.cs
```

### Domain.UnitTests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using {ProjectName}.Domain.Abstractions;
```

### Application.UnitTests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using NSubstitute;
global using {ProjectName}.Domain.Abstractions;
```

### Infrastructure.IntegrationTests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using {ProjectName}.Infrastructure.Data;
```

### API.IntegrationTests/GlobalUsings.cs

```csharp
global using System.Net;
global using System.Net.Http.Json;
global using Xunit;
global using FluentAssertions;
global using Microsoft.Extensions.DependencyInjection;
global using {ProjectName}.Infrastructure.Data;
```

### ArchitectureTests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using NetArchTest.Rules;
```

---

## Step 7 — Starter ArchitectureTests class

Create `tests/{ProjectName}.ArchitectureTests/LayerDependencyTests.cs` to verify the core Clean Architecture rules:

```csharp
namespace {ProjectName}.ArchitectureTests;

public class LayerDependencyTests
{
    private const string DomainNamespace        = "{ProjectName}.Domain";
    private const string ApplicationNamespace   = "{ProjectName}.Application";
    private const string InfrastructureNamespace = "{ProjectName}.Infrastructure";
    private const string ApiNamespace           = "{ProjectName}.API";

    [Fact]
    public void Domain_Should_Not_HaveDependencyOn_Application()
    {
        Types.InAssembly(typeof({ProjectName}.Domain.Abstractions.IRepository<>).Assembly)
             .Should().NotHaveDependencyOn(ApplicationNamespace)
             .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_Should_Not_HaveDependencyOn_Infrastructure()
    {
        Types.InAssembly(typeof({ProjectName}.Domain.Abstractions.IRepository<>).Assembly)
             .Should().NotHaveDependencyOn(InfrastructureNamespace)
             .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_Should_Not_HaveDependencyOn_Infrastructure()
    {
        Types.InAssembly(typeof({ProjectName}.Application.DependencyInjection).Assembly)
             .Should().NotHaveDependencyOn(InfrastructureNamespace)
             .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_Should_Not_HaveDependencyOn_Api()
    {
        Types.InAssembly(typeof({ProjectName}.Application.DependencyInjection).Assembly)
             .Should().NotHaveDependencyOn(ApiNamespace)
             .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

> The assembly entry points above (`IRepository<>`, `DependencyInjection`) are examples — adapt the
> type names to whatever public types exist in those assemblies after running the scaffold.

---

## Step 8 — Verify

```bash
dotnet build
dotnet test --no-build
```

With no user-written tests yet (only the 4 architecture tests), `dotnet test` should report the architecture tests passing and all other projects showing "No tests found". The build must be green before moving on.
