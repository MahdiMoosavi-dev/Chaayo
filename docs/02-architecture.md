# 2. Architecture

## 2.1 Style: modular monolith with clean layers

One deployable ASP.NET Core application, split internally into **modules** (Catalog, Recipes, Inventory, Purchasing, Sales, Pricing, Reporting, Identity, Settings), each with clean layers.

**Why not microservices?** A production or sale must update items, recipes, balances and the ledger **in one database transaction**. Splitting these across services would turn a simple transaction into a distributed one. One cafeteria (or a few) needs no independent scaling. Clear module boundaries keep the option open later.

## 2.2 System context

```mermaid
flowchart LR
    owner(["Owner"])
    sup(["Supervisor"])
    staff(["Staff / Cashier"])
    inventory["Inventory<br/>Stock, pricing and reporting"]
    msg["Messaging (later)<br/>Telegram / email daily summary"]
    excel["Excel / CSV (later)<br/>exports"]

    owner -- "prices, settings, reports" --> inventory
    sup -- "items, recipes, purchases, production, counts" --> inventory
    staff -- "sales, waste" --> inventory
    inventory -. "daily summary, low-stock alerts" .-> msg
    inventory -. "report files" .-> excel
```

## 2.3 Containers

```mermaid
flowchart TB
    subgraph clients ["Clients"]
        web["Admin web app<br/>(React or Blazor, decision Q-8)"]
        pos["Sales screen<br/>tablet browser, same web app"]
    end

    subgraph server ["Server (Docker)"]
        api["Inventory.Api<br/>ASP.NET Core Web API, .NET 10<br/>REST + OpenAPI"]
        worker["Background jobs<br/>(hosted service in the same process)<br/>alerts, balance check, summaries"]
    end

    db[("PostgreSQL 18<br/>all data")]
    backup[("Daily backup<br/>pg_dump, 30 days")]

    web -- "HTTPS JSON, JWT" --> api
    pos -- "HTTPS JSON, JWT" --> api
    api -- "EF Core / Npgsql" --> db
    worker -- "EF Core" --> db
    db -. "nightly" .-> backup
```

## 2.4 Modules and dependencies

```mermaid
flowchart LR
    Catalog["Catalog<br/>items, categories, units"]
    Recipes["Recipes<br/>versions, cycle check, tree"]
    Pricing["Pricing<br/>cost roll-up, suggested price"]
    Inventory["Inventory<br/>balances, operations, ledger"]
    Purchasing["Purchasing<br/>suppliers, purchases"]
    Sales["Sales<br/>sale lines, made-to-order expansion"]
    Reporting["Reporting<br/>read-only queries"]
    Identity["Identity<br/>users, roles, login"]
    Settings["Settings"]

    Recipes --> Catalog
    Pricing --> Recipes
    Pricing --> Inventory
    Inventory --> Catalog
    Inventory --> Recipes
    Purchasing --> Inventory
    Sales --> Inventory
    Sales --> Recipes
    Reporting --> Inventory
    Reporting --> Catalog
    Inventory --> Settings
```

Rules: no cycles between modules. Reporting only reads. **Inventory owns every write to balances and the ledger.** Purchasing, Production and Sales describe *what* should change, and Inventory's posting service applies it.

## 2.5 Layers and projects

```mermaid
flowchart TB
    Api["Inventory.Api<br/>controllers, DTOs, auth, OpenAPI, error mapping"]
    App["Inventory.Application<br/>use cases: ItemService, RecipeService, ProductionService,<br/>SalesService, PurchaseService, ReportQueries"]
    Dom["Inventory.Domain<br/>entities, invariants, domain services<br/>no EF, no ASP.NET"]
    Inf["Inventory.Infrastructure<br/>EF Core DbContext, configurations, migrations,<br/>repositories, report SQL, background jobs"]

    Api --> App
    App --> Dom
    Inf --> App
    Inf --> Dom
    Api -. "DI wiring only" .-> Inf
```

```
Inventory.sln
src/
  Inventory.Domain/          Catalog/ Recipes/ Inventory/ Purchasing/ Pricing/ Common/
  Inventory.Application/     <Module>/ (commands, queries, services, interfaces)
  Inventory.Infrastructure/  Persistence/ (DbContext, Configurations/, Migrations/), Reporting/, Jobs/
  Inventory.Api/             Controllers/, Contracts/ (request and response DTOs), Program.cs
tests/
  Inventory.UnitTests/         domain rules, costing, pricing, cycle check
  Inventory.IntegrationTests/  real PostgreSQL (Testcontainers): transactions, constraints, concurrency
```

## 2.6 Life of a write request

```mermaid
sequenceDiagram
    participant C as Client
    participant Ctl as Controller
    participant S as Application service
    participant Dom as Domain
    participant Db as DbContext / PostgreSQL

    C->>Ctl: POST /api/v1/sales + JWT + Idempotency-Key
    Ctl->>Ctl: authorize role, validate DTO
    Ctl->>S: RecordSale(command)
    S->>Db: client_request_id seen? return the earlier result
    S->>Db: BEGIN, load items and recipes, lock balances
    S->>Dom: build operation, expand made-to-order, issue stock
    Dom-->>S: operation + movements, or DomainException
    S->>Db: SaveChanges, COMMIT
    S-->>Ctl: result
    Ctl-->>C: 201 Created, or ProblemDetails 4xx
```

## 2.7 Technology choices

| Concern | Choice | Reason |
|---|---|---|
| Runtime | .NET 10 (LTS), C# | Your choice. Long-term support until 2028. |
| Web | ASP.NET Core Web API, controllers ([ADR-016](09-decisions.md#adr-016), accepted) | One controller per module, built-in validation, ProblemDetails, OpenAPI. Backend only; the UI is a separate client. |
| Data access | EF Core 10 + Npgsql | Migrations, LINQ, transactions. Raw SQL for heavy reports. |
| Database | PostgreSQL 18 | Strong constraints (partial unique, generated columns, check), `numeric`, `xmin` concurrency, free |
| API docs | Microsoft.AspNetCore.OpenApi + Swagger UI | Built in for .NET 10 |
| Auth | JWT bearer tokens, ASP.NET Core Identity password hasher | Simple, stateless; works for web and tablet |
| Validation | Domain invariants + request DTO validation (FluentValidation or DataAnnotations) | Bad input is stopped at the edge; rules are enforced in the core |
| Logging | Serilog → console + file (structured) | Cheap, searchable |
| Tests | xUnit, Testcontainers (PostgreSQL), FluentAssertions-style assertions | Real database for transaction and constraint tests |
| Packaging | Docker Compose: api + postgres | One command to run anywhere |

## 2.8 Cross-cutting concerns

| Concern | Design |
|---|---|
| **Transactions** | One transaction per command (operation). The application service owns BEGIN and COMMIT. |
| **Concurrency** | Stock writes lock balance rows with `SELECT … FOR UPDATE` in item-id order (no deadlocks). Catalog edits use optimistic `xmin` tokens and return 409 on conflict. |
| **Idempotency** | Write endpoints take an `Idempotency-Key` header, stored as `client_request_id` (unique). A retry returns the original result. |
| **Errors** | Domain errors carry a stable `code` (`stock.insufficient`). The API returns RFC 9457 ProblemDetails with `code` and details (for example the list of shortages). 400 invalid input, 404 not found, 409 concurrency conflict or duplicate (name taken, request replayed with different body), 422 business-rule failure (insufficient stock, circular recipe, item has history). |
| **Time** | Store UTC. Inject `TimeProvider` for testability. Group reports in the business time zone. |
| **Money and quantities** | `decimal` everywhere, with the precision in [data model §3.5](03-data-model.md#35-precision-choices). |
| **IDs** | UUID v7 (time-ordered: good index locality, safe to generate in the client for offline use later). |
| **Security** | Role policies per endpoint (Owner, Supervisor, Staff). HTTPS. The database role used by the app has no UPDATE or DELETE on ledger tables. |
| **Background jobs** | Hosted services: nightly balance = Σ movements check, low-stock and margin alerts, report summaries (later). |
| **Configuration** | `appsettings.json` + environment variables. Secrets come from environment or user-secrets, never from git. |

## 2.9 Deployment

```mermaid
flowchart LR
    dev["Developer machine<br/>docker compose up"] --> img["Container image<br/>inventory-api"]
    img --> host["Server or VPS<br/>Docker"]
    host --> pg[("PostgreSQL volume")]
    host --> bk[("Backup folder or object storage")]
```

Environments: **Development** (local Docker), and **Production** (a single VPS or the cafeteria's PC). Migrations are applied by a deploy step, never automatically at app start in production.
