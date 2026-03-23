---
name: dotnet-find-handler
description: >
  Finds and explains the correct CommandHandler or QueryHandler in a .NET Clean Architecture
  project (MediatR CQRS) when the user mentions a use case, feature, or business function —
  whether to debug it, understand its flow, or modify it.
  Trigger immediately whenever the user names a specific feature, use case, or operation and
  asks why it's broken, how it works, or wants to change it — even if they don't say "handler".
  Example triggers: "the booking feature is broken", "how does the payment flow work",
  "why is GetBookings returning wrong data", "walk me through ReserveBooking", "which handler
  processes order cancellation", "what happens when a user registers", "I need to modify the
  checkout use case", "debug the CreateOrder flow", "tính năng đặt phòng bị lỗi",
  "giải thích flow của ReserveBooking", "use case tạo order làm gì", or any similar request
  about tracing, understanding, or fixing a named feature in a CQRS project.
  Always use this skill before anything else when the user names a specific feature or use case
  and asks why it's broken, how it works, or wants to modify it.
---

# dotnet-find-handler — CQRS Handler Finder & Explainer

When the user mentions a feature or use case, the goal is: **find the right handler, read the
code, and explain it clearly** — giving the user enough context to debug, understand, or modify it.

---

## Step 1 — Locate the project structure

Find the `.sln` or `.slnx` file to identify `{ProjectName}`:
```
src/{ProjectName}.Application/   ← All handlers live here
```

If no `.sln` found, look for an `Application` folder containing entity subfolders
(e.g., `Bookings/`, `Orders/`, `Users/`).

---

## Step 2 — Find the matching handler

From what the user describes (feature name, entity, action, or error symptom), locate the
handler using one of two strategies:

### Strategy A — User names a specific handler / command / query
If the user mentions a specific name (`ReserveBooking`, `CreateOrder`, `GetBookings`):
- Search for `*CommandHandler.cs` or `*QueryHandler.cs` matching that name
- Look inside `src/**/{FeatureName}/` first

### Strategy B — User only describes the feature or entity
If the user says something like "the booking feature" or "the payment use case":
1. Infer the entity involved (`Booking`, `Payment`, `Order`, `User`...)
2. Find the entity folder in the Application layer: `src/**/{EntityPlural}/`
3. List all handlers in that folder, then either ask the user to pick, or infer the best
   match from context

### Fallback — Name doesn't match (Vietnamese, aliases, or vague descriptions)

If no handler is found by direct name match, do **not** give up. Work through these steps:

1. **Scan all handler files** — list every `*CommandHandler.cs` and `*QueryHandler.cs` in the
   Application layer. Read their file names and folder names to build a mental map of what exists.

2. **Read README.md files** — each feature folder contains a `README.md` with business
   descriptions in natural language (possibly in Vietnamese). Search README files for keywords
   that match what the user described.

3. **Match by intent** — use semantic reasoning:
   - "đặt phòng" → look for `Reserve`, `Book`, `Create` in `Bookings/`
   - "hủy đơn" → look for `Cancel` in `Orders/` or `Bookings/`
   - "thanh toán" → look for `Pay`, `Payment`, `Checkout`
   - "đăng ký" → look for `Register`, `SignUp` in `Users/`
   - "danh sách" / "lấy tất cả" → likely a `GetAll*QueryHandler`

4. **Present candidates** — if you find plausible matches but aren't certain, show them:
   > "I couldn't find an exact match for 'đặt phòng'. These handlers look related:
   > - `ReserveBookingCommandHandler` (Bookings/ReserveBooking/)
   > - `CreateBookingCommandHandler` (Bookings/CreateBooking/)
   >
   > Which one do you mean, or should I read both?"

5. **Ask for clarification** as a last resort — only if the above steps yield nothing useful.

**When multiple handlers exist for the same entity**, list them and ask:
> "I found these handlers for `Bookings`:
> - `ReserveBookingCommandHandler` — creates a new booking
> - `CancelBookingCommandHandler` — cancels a booking
> - `GetBookingQueryHandler` — fetches a single booking
> - `GetBookingsQueryHandler` — fetches a list of bookings
>
> Which one are you referring to?"

---

## Step 3 — Read the handler and related files

Once the handler is identified, read these files **in parallel**:

1. **README.md** — business documentation for this use case (requirements, rules, notes)
2. **Handler** — `{OperationName}CommandHandler.cs` or `{OperationName}QueryHandler.cs`
3. **Command / Query** — `{OperationName}Command.cs` / `{OperationName}Query.cs` (input shape)
4. **Validator** (commands only) — `{OperationName}CommandValidator.cs`
5. **Response DTO** (if present) — `{OperationName}Response.cs`

Read README.md first — it explains the business intent behind the use case, which helps
interpret the implementation correctly. If the README and the code diverge, flag it to the user.

If the handler calls a method on a domain entity (e.g., `booking.Reserve(...)`,
`order.Submit()`), also read that method in the Domain entity — this is where the real
business rules live.

---

## Step 4 — Present findings

Tailor the depth and focus to what the user actually needs:

### When the user is debugging / has an error

Focus on the failure points:
- Validation (Validator) — does it fail early before the handler runs?
- Business rule in the entity method — what conditions can cause failure?
- Repository calls — is data fetched correctly?
- `Result` return values — what errors does the handler return and when?

Example format:
```
Handler: ReserveBookingCommandHandler
Input: ReserveBookingCommand { ApartmentId, UserId, StartDate, EndDate }

Validation (FluentValidation):
  ✓ StartDate < EndDate
  ✓ ApartmentId not empty

Execution flow:
  1. Fetch User       → not found → Result.Failure(UserErrors.NotFound)
  2. Fetch Apartment  → not found → Result.Failure(ApartmentErrors.NotFound)
  3. Check overlap    → conflict  → Result.Failure(BookingErrors.Overlap)
  4. Call Booking.Reserve(...) on the entity
  5. SaveChangesAsync

→ If bookings fail despite valid dates, check step 3 (overlap logic) or step 4 (entity method)
```

### When the user wants to understand the flow

Explain the full end-to-end flow:
- Where input comes from (API endpoint → Command/Query)
- What the handler does, step by step
- Which domain entity methods are called and why
- What the response looks like

### When the user wants to modify the feature

Point out exactly which file to change for each type of modification:
- Changing input → edit the Command/Query
- Changing a business rule → edit the Entity method
- Changing the response shape → edit the Response DTO
- Adding validation → edit the Validator
- Adding a new dependency → update handler constructor injection

---

## Important Reminders

- **Read the actual code before explaining** — don't infer from file names alone, implementation
  often differs from what the name suggests
- **Real business logic lives in the Entity method**, not the handler — when debugging, always
  read the entity method being called
- **The handler is just orchestration** — it calls domain, calls repositories, saves, and returns
  a result
- If no matching handler is found, ask the user for more detail: entity name, operation name,
  or folder path

---

## Related skills

- **dotnet-clean-feature** — scaffold a new Command/Query + Handler
- **dotnet-clean-architect** — review layer violations in existing code
- **dotnet-clean-endpoint** — find the Minimal API endpoint that calls this handler
