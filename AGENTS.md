# AGENTS.md — Chaayo (Cafeteria Inventory & Item Composition System)

Instructions for AI coding agents (Claude Code, Codex, Copilot, Cursor, …) working in this repo.
The source design is [cafeteria_inventory_system_design.docx](cafeteria_inventory_system_design.docx) (v0.1, 2026-11-15).
If this file and the design doc disagree, ask; do not silently pick one.

## Project summary

Backend for a cafeteria inventory system. Every stockable thing (flour, sugar, cake, piece of cake,
bottle of tea, "Tea + Cake" bundle) is one `Item`. Items can be composed of other items through a
self-referencing `ItemComposition` table (nested to any depth). Producing a composite item deducts
its components from stock and adds the finished item, atomically, with an audit trail.

## Tech stack

- .NET 10 SDK, C#, ASP.NET Core Web API (controllers)
- EF Core with PostgreSQL (Npgsql); migrations via `dotnet ef`
- OpenAPI via `Microsoft.AspNetCore.OpenApi` + Swagger UI
- xUnit for tests; integration tests run against a real PostgreSQL (Testcontainers), not the EF in-memory provider (it does not enforce transactions or constraints)

## Solution layout (modular monolith)

```
Chaayo.sln
src/
  Chaayo.Api/            # Controllers, DTOs, request validation, DI wiring, OpenAPI
  Chaayo.Application/    # Use cases / services: ItemService, CompositionService, InventoryService, ProductionService
  Chaayo.Domain/         # Entities, value objects, domain rules (no EF, no ASP.NET references)
  Chaayo.Infrastructure/ # EF Core DbContext, entity configurations, migrations, repositories
tests/
  Chaayo.UnitTests/
  Chaayo.IntegrationTests/
```

Dependency direction: Api → Application → Domain; Infrastructure → Application/Domain. Domain depends on nothing.
Keep controllers thin: no business logic or DbContext access in controllers.

## Commands

```bash
dotnet build
dotnet test
dotnet run --project src/Chaayo.Api
dotnet ef migrations add <Name> --project src/Chaayo.Infrastructure --startup-project src/Chaayo.Api
dotnet ef database update --project src/Chaayo.Infrastructure --startup-project src/Chaayo.Api
```

## Domain model

| Entity | Key fields |
|---|---|
| `Item` | `Id`, `Name`, `Description?`, `Unit`, `IsActive`, `CreatedAt`, `UpdatedAt` |
| `ItemComposition` | `Id`, `ParentItemId` → Item, `ComponentItemId` → Item, `Quantity` (per 1 parent), timestamps |
| `StockBalance` | `ItemId` → Item, `Quantity`, concurrency token |
| `InventoryMovement` | `Id`, `ItemId`, `QuantityDelta` (+/−), `Type` (Production, Purchase, Sale, Adjustment…), `OperationId`, `CreatedAt`, `CreatedBy` |

## Invariants (enforce in code AND in the database where possible)

1. `ItemComposition.Quantity > 0` (check constraint).
2. Unique `(ParentItemId, ComponentItemId)`.
3. `ParentItemId != ComponentItemId` (check constraint).
4. No circular compositions (A → B → … → A). Validate in `CompositionService` before saving, by walking the component graph.
5. Bundles reuse existing component items; never duplicate an item to use it as a component.
6. A production operation runs in **one database transaction**: deduct every component, add the
   finished item, and write all `InventoryMovement` rows with a shared `OperationId`. Any failure rolls back everything.
7. Use optimistic concurrency (or row locks) on `StockBalance` so concurrent productions can't oversell.
8. Use `decimal` for quantities, never `float`/`double`.
9. Stock balances change only through movements; never update a balance without writing a movement.

## Open business rules (do not guess, ask the user)

- Virtual bundle vs. physically stocked composite: when does "Tea + Cake" deduct stock (at creation or at sale)?
- Can composite items hold their own stock balance?
- Composition versioning when a recipe is edited after production.
- Negative stock allowed, or reject when insufficient? (Default until decided: **reject**.)
- Unit set and conversions (g/kg, ml/l, piece, bottle).
- Whether compositions are recipes (BOM), bundles, or both. This may need an item type/behaviour field.

## Conventions

- Nullable reference types and `TreatWarningsAsErrors` on.
- Async all the way (`async`/`await`, pass `CancellationToken`).
- Configure entities with `IEntityTypeConfiguration<T>`, not data annotations.
- Return `ProblemDetails` for errors; map domain errors (e.g. `InsufficientStockException`, `CircularCompositionException`) to 4xx.
- Timestamps in UTC (`DateTimeOffset`).
- Every new service method gets tests; production logic needs tests for nested compositions, insufficient stock, rollback, and the audit trail.
- Commit messages: Conventional Commits (`feat:`, `fix:`, `chore:` …).
