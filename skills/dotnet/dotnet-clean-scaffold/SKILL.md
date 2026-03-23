---
name: dotnet-clean-scaffold
description: >
  Scaffold a new ASP.NET Core Clean Architecture solution from scratch using the dotnet CLI.
  Creates a complete 4-layer solution (Domain / Application / Infrastructure / API) with all
  NuGet packages and prerequisite abstractions that dotnet-clean-entity, dotnet-clean-repository,
  dotnet-clean-feature, and dotnet-clean-endpoint depend on. Use this skill whenever the user
  wants to start a new .NET project, create a new solution, scaffold a Clean Architecture project,
  or bootstrap a new .NET Web API from zero. Trigger on: "tạo project mới", "tạo solution",
  "scaffold project", "tạo clean architecture project", "khởi tạo project .NET", "tạo API mới",
  "tạo project từ đầu", "bootstrap .NET", "setup project", "tạo project X" — even if the user
  just names the project without saying "scaffold". Always use this skill before any other
  dotnet-clean-* skill when no solution file exists yet.
---

# .NET 10 Clean Architecture — Project Scaffolder

Creates a complete solution with 4 layers (Domain / Application / Infrastructure / API),
NuGet packages, and all base abstractions that the other dotnet-clean-* skills depend on.
Uses real `dotnet` CLI commands — no manual file creation needed.

## Reference files

| File | Contents |
|------|----------|
| `references/msbuild-props.md` | Directory.Build.props, Directory.Packages.props, cleaned .csproj templates |
| `references/domain-abstractions.md` | BaseEntity, IRepository, IUnitOfWork, Result, Error, ICommand, IQuery |
| `references/application-abstractions.md` | ICommandHandler, IQueryHandler, ISqlConnectionFactory, ValidationBehavior, DI |
| `references/infrastructure-foundation.md` | AppDbContext, BaseEntityConfiguration, Repository\<T\>, SqlConnectionFactory, DI |
| `references/api-foundation.md` | IEndpoint, EndpointExtensions, Program.cs, appsettings.json |

---

## Step 0 — Gather inputs

Ask for (or infer from context):
- **Project name** — `{ProjectName}` in PascalCase (e.g., `Bookings`, `EcommerceApi`)
- **Output directory** — where to create the solution folder (default: current directory)
- **Database** — PostgreSQL (default) or SQL Server?

If the user gives the project name in Vietnamese, translate to English PascalCase and confirm.
Confirm all inputs before running any commands.

---

## Step 1 — Create solution and projects

Run these commands **in sequence** from the output directory.
Replace `{ProjectName}` with the actual name throughout.

```bash
mkdir {ProjectName}
cd {ProjectName}

dotnet new classlib -n {ProjectName}.Domain    -o src/{ProjectName}.Domain    --framework net10.0
dotnet new classlib -n {ProjectName}.Application -o src/{ProjectName}.Application --framework net10.0
dotnet new classlib -n {ProjectName}.Infrastructure -o src/{ProjectName}.Infrastructure --framework net10.0
dotnet new webapi   -n {ProjectName}.API       -o src/{ProjectName}.API       --framework net10.0

dotnet add src/{ProjectName}.Application    reference src/{ProjectName}.Domain
dotnet add src/{ProjectName}.Infrastructure reference src/{ProjectName}.Application
dotnet add src/{ProjectName}.Infrastructure reference src/{ProjectName}.Domain
dotnet add src/{ProjectName}.API            reference src/{ProjectName}.Infrastructure
dotnet add src/{ProjectName}.API            reference src/{ProjectName}.Application
```

Then create **`{ProjectName}.slnx`** at the solution root. `.slnx` is a simple XML format — write it directly:

```xml
<Solution>
  <Project Path="src/{ProjectName}.Domain/{ProjectName}.Domain.csproj" />
  <Project Path="src/{ProjectName}.Application/{ProjectName}.Application.csproj" />
  <Project Path="src/{ProjectName}.Infrastructure/{ProjectName}.Infrastructure.csproj" />
  <Project Path="src/{ProjectName}.API/{ProjectName}.API.csproj" />
</Solution>
```

> `.slnx` is the modern XML-based solution format (introduced in .NET 9). It replaces the legacy
> `.sln` text format with a clean, human-readable XML that's easy to diff and merge.
> `dotnet build`, `dotnet restore`, and IDE tooling all support it.

---

## Step 2 — MSBuild props (Central Package Management)

Read `references/msbuild-props.md` and create two files at the **solution root**:

1. **`Directory.Build.props`** — shared compiler settings for every project (Nullable, ImplicitUsings, LangVersion)
2. **`Directory.Packages.props`** — Central Package Management (CPM): all NuGet versions defined once, referenced by name only in `.csproj` files

> Why CPM? With `ManagePackageVersionsCentrally=true`, every `<PackageReference>` in every `.csproj`
> omits the `Version` attribute. Versions live exclusively in `Directory.Packages.props`.
> Upgrading a package means editing one line, not hunting across 4 project files.

Do **not** run `dotnet add package` — packages are added by editing `.csproj` files directly (Step 3).

---

## Step 3 — Clean .csproj files and add package references

Read `references/msbuild-props.md` for the exact `.csproj` templates. For each project:

1. Open the generated `.csproj`
2. Remove properties that are now inherited from `Directory.Build.props` (`<Nullable>`, `<ImplicitUsings>`)
3. Add `<PackageReference Include="PackageName" />` entries — **no Version attribute** (version comes from CPM)
4. For the API project, also remove `Version` from the `<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="..." />` that `dotnet new webapi` generates

---

## Step 4 — Delete generated boilerplate

```bash
rm src/{ProjectName}.Domain/Class1.cs
rm src/{ProjectName}.Application/Class1.cs
rm src/{ProjectName}.Infrastructure/Class1.cs
rm -f src/{ProjectName}.API/WeatherForecast.cs
```

---

## Step 5 — Create folder structure

Create these directories (files go in the next steps):

```
src/{ProjectName}.Domain/
  Abstractions/
  Entities/
  Enums/

src/{ProjectName}.Application/
  Abstractions/
    Data/
    Messaging/
  Behaviors/

src/{ProjectName}.Infrastructure/
  Data/
    Configurations/
  Repositories/

src/{ProjectName}.API/
  Endpoints/
  Extensions/
```

On bash: `mkdir -p src/{ProjectName}.Domain/{Abstractions,Entities,Enums}`
On PowerShell: create each with `New-Item -ItemType Directory`.

---

## Step 6 — Domain layer

Read `references/domain-abstractions.md` and create all files listed there.

---

## Step 7 — Application layer

Read `references/application-abstractions.md` and create all files listed there.

---

## Step 8 — Infrastructure layer

Read `references/infrastructure-foundation.md` and create all files listed there.
Substitute the correct EF Core provider call:
- PostgreSQL → `optionsBuilder.UseNpgsql(connectionString)`
- SQL Server → `optionsBuilder.UseSqlServer(connectionString)`

---

## Step 9 — API layer

Read `references/api-foundation.md` and create all files listed there.

---

## Step 10 — Verify

```bash
dotnet build
```

Fix any compilation errors before marking this task done. The build must be green.

---

## What the user can do next

Once the scaffold is complete, the following skills work immediately:
- **dotnet-clean-entity** — add domain entities with rich DDD patterns
- **dotnet-clean-repository** — add EF Core configuration + repository for an entity
- **dotnet-clean-feature** — add MediatR Command / Query + Handler
- **dotnet-clean-endpoint** — add Minimal API endpoints

---

## Important notes

- Target **.NET 10** (`net10.0`) and **C# 13** throughout.
- Use **file-scoped namespaces** everywhere (`namespace Foo.Bar;` not `namespace Foo.Bar { }`).
- The `.slnx` file is created at the solution root (XML-based format, .NET 9+). All source projects live under `src/`.
- Connection string placeholder goes in `appsettings.Development.json` — never commit real credentials.
- Run `dotnet build` after Step 1 to catch project reference issues early; run again at Step 9 for the full check.
