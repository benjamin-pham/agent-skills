# README.md Template for Operation Folders

Every Command/Query operation folder contains a `README.md` documenting its business context.

## Template

```markdown
# {OperationName}

## Description

{Brief business description: what this operation does, why it exists, the business flow or use case it serves.}

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| {Name} | {Type} | Yes/No | {What this parameter represents} |

## Output

| Field | Type | Description |
|-------|------|-------------|
| {Name} | {Type} | {What this field represents} |

> For Commands that return only success/failure (e.g., `Result<Guid>`), replace the table with a single line describing the return value, e.g.: "Returns the `Guid` of the newly created booking."

## Validation Rules

| Rule | Description |
|------|-------------|
| {Field} is required | {Detail or business reason} |
| {Field} max length {N} | {Detail or business reason} |
| {Custom rule} | {Detail or business reason} |

> For Queries (no validator), list any handler-level checks here (e.g., "Returns NotFound if entity does not exist"). If none, write "No validation rules."

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | {YYYY-MM-DD} | Initial creation |
```

## Example — ReserveBooking Command

```markdown
# ReserveBooking

## Description

Reserves a booking for an apartment within a specified date range. Validates that the dates don't overlap with existing bookings, calculates the total price including cleaning and service fees, and persists the new booking with `Reserved` status.

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| ApartmentId | Guid | Yes | The apartment to reserve |
| UserId | Guid | Yes | The user making the reservation |
| StartDate | DateOnly | Yes | Check-in date |
| EndDate | DateOnly | Yes | Check-out date |

## Output

Returns the `Guid` of the newly created booking.

## Validation Rules

| Rule | Description |
|------|-------------|
| ApartmentId is required | Must reference an existing apartment |
| UserId is required | Must reference an existing user |
| StartDate < EndDate | Check-in must be before check-out |
| No overlapping bookings | Business rule enforced in handler via domain service |

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-15 | Initial creation |
| 1.1 | 2025-03-20 | Added cleaning fee to price calculation |
```

## Example — GetBooking Query

```markdown
# GetBooking

## Description

Retrieves booking details by ID, including apartment info and pricing breakdown. Used by the booking detail page.

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| BookingId | Guid | Yes | The booking to retrieve |

## Output

| Field | Type | Description |
|-------|------|-------------|
| Id | Guid | Booking identifier |
| ApartmentName | string | Name of the booked apartment |
| StartDate | DateOnly | Check-in date |
| EndDate | DateOnly | Check-out date |
| TotalPrice | decimal | Final price including all fees |
| Status | string | Current booking status |

## Validation Rules

Returns `NotFound` error if booking does not exist.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-15 | Initial creation |
| 1.1 | 2025-04-10 | Added Status field to response |
```

## Rules for Updating

When modifying an existing Command/Query that already has a `README.md`:

1. **Update** the Description, Input, Output, and Validation Rules sections to reflect the current state
2. **Append** a new row to the Version History table — increment the version, use today's date, and summarize the change
3. **Never delete** previous Version History entries
