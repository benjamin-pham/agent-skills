---
name: dotnet-clean-feature
description: >
  Generate MediatR CQRS features — individual Commands or Queries with Handlers, Validators,
  and Response DTOs — for an ASP.NET Core Clean Architecture project following
  Pragmatic Clean Architecture patterns. Uses vertical slice folder structure,
  custom Result pattern, FluentValidation, Dapper for queries (CQRS read side),
  and EF Core repositories for commands (write side).
  Use this skill whenever the user wants to create/add/generate a command, query,
  or handler in a .NET clean architecture project.
  Trigger on phrases like "tạo feature", "thêm command", "add query",
  "thêm use case", "create feature for Product", "sinh command CreateProduct",
  "add GetAll query", "tạo handler", or any Vietnamese/English request about adding
  MediatR commands, queries, handlers, validators, or application-layer features.
  Also trigger as part of "tạo API cho entity X" — this skill handles the Application layer
  (pair with dotnet-clean-endpoint for the API layer).
metadata:
  related-skills:
    - dotnet-clean-architect
    - dotnet-clean-endpoint
    - dotnet-clean-entity
    - dotnet-clean-repository
---

# ASP.NET Core — Clean Architecture Command & Query Generator (MediatR CQRS)

Generates individual MediatR Commands (with Handlers + Validators) and Queries (with Handlers + Response DTOs) for a Clean Architecture .NET project, following CQRS with Dapper (read) and Repository + UnitOfWork (write).

## Prerequisites

Before generating features, ensure:
1. The entity already exists in Domain layer
2. `ICommand`, `IQuery`, `ICommandHandler`, `IQueryHandler` exist in `Application/Abstractions/Messaging/`
3. `Result<T>` and `Error` exist in `Domain/Abstractions/`
4. `ISqlConnectionFactory` exists in `Application/Abstractions/Data/`
5. `IUnitOfWork` exists in `Domain/Abstractions/`
6. ValidationBehavior pipeline is registered in `DependencyInjection.cs`

If any are missing, inform the user which prerequisite is needed before proceeding.

## Detect Project Structure

1. Find the `.sln` or `.slnx` file to identify `{ProjectName}`
2. Confirm Application project exists at `src/{ProjectName}.Application/`
3. Check for existing features to match the established style

---

## Folder Structure — Vertical Slice

Features are organized by entity, then by operation:

```
src/{ProjectName}.Application/
  Features/
    {EntityPlural}/                          # e.g., Bookings/
      {OperationName}/                       # e.g., ReserveBooking/
        README.md                            # Business documentation
        {OperationName}Command.cs            # or {OperationName}Query.cs
        {OperationName}CommandHandler.cs     # or {OperationName}QueryHandler.cs
        {OperationName}CommandValidator.cs   # Commands only
        {OperationName}Response.cs           # if operation returns a DTO
```

**Naming rules:**
- Folder and file names use the **operation name**, not generic CRUD names
- Commands: `{Verb}{Entity}Command` (e.g., `ReserveBookingCommand`, `RegisterUserCommand`)
- Queries: `{Verb}{Entity}Query` (e.g., `GetBookingQuery`, `SearchApartmentsQuery`)
- Use **domain language** — prefer `Reserve` over `Create`, `Search` over `GetAll` when it fits the domain

**Simple features** (single query, no commands): files can sit directly in `Features/{EntityPlural}/` without operation subfolder, matching the Apartments pattern in the reference project.

---

## Generating a Command

When user asks to add a command (create, update, delete, or any write operation):

Read `references/command-template.md` and generate:

1. **Command record** — `ICommand<TResponse>` with required properties
2. **Command Handler** — `ICommandHandler<TCommand, TResponse>`, `internal sealed class`
   - Inject repositories via constructor (not primary constructor — use explicit fields)
   - Inject `IUnitOfWork` for persistence
   - Return `Result<T>` — use `Result.Failure<T>(EntityErrors.X)` for errors
   - Call `_unitOfWork.SaveChangesAsync()` instead of repo's own save
3. **Command Validator** — `AbstractValidator<TCommand>`, `internal class`
   - Validate all required fields with `RuleFor`
   - Add business rules that can be checked without DB access

## Generating a Query

When user asks to add a query (get by id, get all, search, list):

Read `references/query-template.md` and generate:

1. **Query record** — `IQuery<TResponse>` with filter/id parameters
   - For cached queries: use `ICachedQuery<TResponse>` with `CacheKey` and `Expiration`
2. **Query Handler** — `IQueryHandler<TQuery, TResponse>`, `internal sealed class`
   - Inject `ISqlConnectionFactory` (and `IUserContext` if authorization needed)
   - Use **Dapper with raw SQL** — do NOT use EF Core or repositories for reads
   - Map snake_case DB columns to PascalCase DTO properties via SQL aliases
   - Return `Result<T>`
3. **Response DTO** — `sealed class` with `{ get; init; }` properties
   - Place in the same folder as the query, or at entity level if shared across queries

---

## README.md — Business Documentation

Every operation folder **must** contain a `README.md` that documents the business context. Read `references/readme-template.md` for the full template.

### When creating a new Command/Query

Generate `README.md` alongside the code files with:
- **Business description** — what this operation does, why it exists, the business flow
- **Input** — request parameters with types and descriptions
- **Output** — response fields with types and descriptions
- **Validation rules** — all rules enforced by the Validator (commands) or by the handler logic
- **Version history** — initial entry with creation date

### When updating an existing Command/Query

If the operation folder already has a `README.md`:
1. Update the relevant sections (description, input/output, validation) to reflect the changes
2. Append a new entry to the **Version History** section at the bottom with: version number, date, and summary of what changed

If `README.md` is missing in an existing operation folder, create it from scratch following the template.

---

## Important Reminders

Read `references/conventions.md` for the full list. Key rules:

- **File-scoped namespaces** everywhere
- **`internal sealed class`** for all handlers
- **`internal class`** for validators
- **`sealed record`** for commands and queries
- **`sealed class`** for response DTOs
- Handlers use **explicit constructor + private readonly fields** (not primary constructors)
- Commands go through **Repository + UnitOfWork** (write side)
- Queries go through **Dapper + ISqlConnectionFactory** (read side) — this is CQRS
- All SQL uses **snake_case** column names with PascalCase aliases — see `dotnet-clean-repository/references/snake_case_rules.md` for the complete naming rules
- **No AutoMapper/Mapster** — Dapper maps directly to Response DTOs via SQL aliases
- Validators only on **Commands** (not queries)
- **CancellationToken** — every `Handle` method signature must include `CancellationToken cancellationToken`, and pass it down to every async call (`GetByIdAsync`, `SaveChangesAsync`, Dapper queries, etc.)
- When user provides operation names in Vietnamese, translate to English PascalCase and confirm
