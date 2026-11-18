# AGENTS.md — Inventory (Cafeteria Stock, Recipes & Pricing System)

Instructions for AI coding agents (Claude Code, Codex, Copilot, Cursor, …) working in this repo.

## Current phase: DESIGN REVIEW

The system design lives in [docs/](docs/README.md) (v0.2). **Do not write implementation code until the owner approves the design.**
Changes to the design go into the docs, with a new or updated entry in [docs/09-decisions.md](docs/09-decisions.md).
The original v0.1 design is [cafeteria_inventory_system_design.docx](cafeteria_inventory_system_design.docx). Where it and `docs/` disagree, `docs/` wins.

## Source of truth

| Topic | Document |
|---|---|
| Requirements, glossary, roles | [docs/01-requirements.md](docs/01-requirements.md) |
| Architecture, layers, tech stack | [docs/02-architecture.md](docs/02-architecture.md) |
| **Data structure: tables, constraints** | [docs/03-data-model.md](docs/03-data-model.md) |
| Domain model, invariants (INV-xx) | [docs/04-domain-model.md](docs/04-domain-model.md) |
| Business processes, costing, pricing | [docs/05-business-processes.md](docs/05-business-processes.md) |
| Reports and KPI formulas | [docs/06-reporting.md](docs/06-reporting.md) |
| API | [docs/07-api.md](docs/07-api.md) |
| Roadmap and MVP build order | [docs/08-features-roadmap.md](docs/08-features-roadmap.md) |
| Decisions (ADR-xxx) | [docs/09-decisions.md](docs/09-decisions.md) |

## Tech stack

.NET 10 · C# · ASP.NET Core Web API (controllers) · EF Core 10 + Npgsql · PostgreSQL 18 · xUnit + Testcontainers (real PostgreSQL; never the EF in-memory provider).

## Solution layout

```
src/Inventory.Domain          entities, invariants, domain services (no EF, no ASP.NET)
src/Inventory.Application     use cases per module
src/Inventory.Infrastructure  EF Core, configurations, migrations, report SQL, jobs
src/Inventory.Api             controllers, DTOs, auth, OpenAPI
tests/Inventory.UnitTests, tests/Inventory.IntegrationTests
```

Dependencies: Api → Application → Domain; Infrastructure → Application/Domain. Thin controllers.

## Commands

```bash
dotnet build
dotnet test
dotnet run --project src/Inventory.Api
dotnet ef migrations add <Name> --project src/Inventory.Infrastructure --startup-project src/Inventory.Api
```

## Non-negotiable rules (details in the docs)

1. One `items` table; composition goes through versioned `recipes` + `recipe_lines` (parent → component).
2. Stock changes only by appending `stock_movements` under an `inventory_operations` header, in **one transaction**, updating `stock_balances` in the same transaction. Never UPDATE or DELETE ledger rows; fix mistakes with a reversal.
3. Lock balance rows `FOR UPDATE` in item-id order inside stock operations.
4. Weighted average cost; production output cost = cost of the components consumed.
5. Reject negative stock unless `business_settings.allow_negative_stock`.
6. No circular recipes (graph check in the application; self-reference also blocked by a DB check).
7. `decimal` for all money and quantities; UTC timestamps; UUID v7 keys.
8. Archive items with history; hard-delete only items without history.
9. Every INV-xx rule in docs/04 gets a test; error codes match the catalogue.

## Conventions

Nullable on, warnings as errors, async + `CancellationToken`, `IEntityTypeConfiguration<T>` (not annotations), ProblemDetails with a stable `code`, Conventional Commits.
