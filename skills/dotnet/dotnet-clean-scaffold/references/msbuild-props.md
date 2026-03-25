# MSBuild Props — Central Package Management

Create both files at the **solution root** (same level as the `.slnx` file).

---

## Directory.Build.props

Applies to **every project** in the solution automatically. Centralizes quality settings
so individual `.csproj` files don't repeat them.

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

Because these are inherited, remove `<Nullable>`, `<ImplicitUsings>` from each generated `.csproj`.

---

## Directory.Packages.props

Enables **Central Package Management** — versions defined once, referenced by name only.

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup Label="Shared">    
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="10.0.5" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Abstractions" Version="10.0.5" />
  </ItemGroup>

  <ItemGroup Label="Application">
    <PackageVersion Include="MediatR" Version="11.1.0" />
    <PackageVersion Include="FluentValidation" Version="12.1.1" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="12.1.1" />
  </ItemGroup>

  <ItemGroup Label="Infrastructure">
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.1" />
    <PackageVersion Include="Dapper" Version="2.1.72" />
  </ItemGroup>

  <ItemGroup Label="API">
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="9.0.0" />
    <PackageVersion Include="Scalar.AspNetCore" Version="2.13.13" />
  </ItemGroup>

  <ItemGroup Label="Testing">
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="3.0.1" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="FluentAssertions" Version="7.2.2" />
    <PackageVersion Include="coverlet.collector" Version="6.0.4" />
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.0" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="3.10.0" />
    <PackageVersion Include="Npgsql" Version="10.0.2" />
    <PackageVersion Include="Respawn" Version="6.2.1" />
    <PackageVersion Include="NetArchTest.Rules" Version="1.3.2" />
  </ItemGroup>

</Project>
```

> in `Directory.Packages.props` — having a version entry doesn't add a package, it just makes the
> version available. Only projects that actually reference the package will pull it in.

---

## Cleaned .csproj templates

After creating the props files, edit each generated `.csproj` to match these templates.
Key rules:
- No `<Nullable>` or `<ImplicitUsings>` (inherited from Directory.Build.props)
- No `Version` attribute on `<PackageReference>` (managed by Directory.Packages.props)

### {ProjectName}.Domain.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <ItemGroup>
    <PackageReference Include="MediatR" />
  </ItemGroup>

</Project>
```

### {ProjectName}.Application.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <ItemGroup>
    <PackageReference Include="MediatR" />
    <PackageReference Include="FluentValidation" />
    <PackageReference Include="FluentValidation.DependencyInjectionExtensions" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
  </ItemGroup>

</Project>
```

### {ProjectName}.Infrastructure.csproj — PostgreSQL

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" />
    <PackageReference Include="Dapper" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Abstractions" />
  </ItemGroup>

</Project>
```

### {ProjectName}.Infrastructure.csproj — SQL Server

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
    <PackageReference Include="Dapper" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Abstractions" />
  </ItemGroup>

</Project>
```

### {ProjectName}.API.csproj

`dotnet new webapi` generates `<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />`.
With CPM enabled, remove the `Version` attribute:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" />
    <PackageReference Include="Serilog.AspNetCore" />
    <PackageReference Include="Scalar.AspNetCore" />
  </ItemGroup>

</Project>
```

> Note: The webapi template may include `<Nullable>` and `<ImplicitUsings>` explicitly — that's OK,
> they match `Directory.Build.props` so there's no conflict. You can remove the duplicates to keep
> the file clean, or leave them — the result is the same.
