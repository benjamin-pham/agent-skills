# Test Project Setup

## Directory Structure

```
{SolutionRoot}/
├── src/
│   ├── {ProjectName}.Domain/
│   ├── {ProjectName}.Application/
│   ├── {ProjectName}.Infrastructure/
│   └── {ProjectName}.API/
├── tests/
│   ├── {ProjectName}.Domain.Tests/
│   └── {ProjectName}.Application.Tests/
├── Directory.Build.props
└── Directory.Packages.props
```

## Step 1 — Add Test Packages to Directory.Packages.props

Add this `<ItemGroup>` to the existing `Directory.Packages.props` at solution root:

```xml
<ItemGroup Label="Testing">
  <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
  <PackageVersion Include="xunit" Version="2.9.3" />
  <PackageVersion Include="xunit.runner.visualstudio" Version="3.0.1" />
  <PackageVersion Include="NSubstitute" Version="5.3.0" />
  <PackageVersion Include="FluentAssertions" Version="8.0.1" />
  <PackageVersion Include="coverlet.collector" Version="6.0.4" />
</ItemGroup>
```

Check NuGet for latest stable versions if the ones above are outdated.

## Step 2 — Create Test Projects

### Domain.Tests

```bash
dotnet new xunit -n {ProjectName}.Domain.Tests -o tests/{ProjectName}.Domain.Tests
dotnet sln add tests/{ProjectName}.Domain.Tests
dotnet add tests/{ProjectName}.Domain.Tests reference src/{ProjectName}.Domain
```

### Application.Tests

```bash
dotnet new xunit -n {ProjectName}.Application.Tests -o tests/{ProjectName}.Application.Tests
dotnet sln add tests/{ProjectName}.Application.Tests
dotnet add tests/{ProjectName}.Application.Tests reference src/{ProjectName}.Application
```

If Application references Domain (which it should in Clean Architecture), Application.Tests can also access Domain types through the transitive reference.

## Step 3 — Update .csproj Files

After creating via `dotnet new xunit`, the .csproj files will have explicit versions. Update them to use Central Package Management (no `Version` attribute) and remove properties already in `Directory.Build.props`.

### {ProjectName}.Domain.Tests.csproj

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

Domain.Tests does NOT need NSubstitute — domain entities are tested directly without mocking.

### {ProjectName}.Application.Tests.csproj

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

Application.Tests includes NSubstitute for mocking repository interfaces and other dependencies.

## Step 4 — Clean Up Generated Files

`dotnet new xunit` creates a default `UnitTest1.cs` and a `GlobalUsings.cs`. Handle them:

1. **Delete `UnitTest1.cs`** — we'll create proper test classes
2. **Update `GlobalUsings.cs`** with project-wide usings:

### Domain.Tests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using {ProjectName}.Domain.Entities;
global using {ProjectName}.Domain.Exceptions;
```

### Application.Tests/GlobalUsings.cs

```csharp
global using Xunit;
global using FluentAssertions;
global using NSubstitute;
global using {ProjectName}.Domain.Entities;
global using {ProjectName}.Domain.Interfaces;
```

## Step 5 — Verify Setup

```bash
dotnet build
dotnet test --no-build
```

Both commands should succeed with 0 tests found (since we deleted the default test file). This confirms the project structure and package references are correct.
