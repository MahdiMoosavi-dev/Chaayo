# 9. Decision Log

Short architecture decision records. Status: **Proposed** until you approve the design; then **Accepted**. To disagree, comment on the ID.

<a id="adr-001"></a>
### ADR-001 Modular monolith
**Decision:** one ASP.NET Core app with internal modules and clean layers.
**Why:** stock operations must be single database transactions, and the load is small.
**Alternatives rejected:** microservices (distributed transactions for every sale).
**Status:** Proposed

<a id="adr-002"></a>
### ADR-002 One `items` table for everything
**Decision:** ingredients, products and bundles are all `items`, distinguished by `type` (Simple or Composite) and `stock_mode` (Stocked or MadeToOrder).
**Why:** from v0.1. Recipes can then point at any item, and reports treat everything alike.
**Status:** Proposed (kept from v0.1)

<a id="adr-003"></a>
### ADR-003 Versioned recipes instead of a flat `ItemComposition`
**Decision:** `recipes` (item, version, yield, is_active) + `recipe_lines` (parent, component, quantity). Recipes are immutable; an edit creates a new version.
**Why:** past production must show the recipe actually used, and a yield handles batches.
**Trade-off:** one extra table, and the UI must hide versioning behind a simple "Edit".
**Status:** Proposed

<a id="adr-004"></a>
### ADR-004 Append-only ledger with operation headers
**Decision:** `inventory_operations` (header) + `stock_movements` (lines). Never updated or deleted. Mistakes are fixed by reversal.
**Why:** a full audit trail; any past report can be reproduced; balances can be verified.
**Status:** Proposed

<a id="adr-005"></a>
### ADR-005 Sale lines separate from stock movements
**Decision:** revenue and COGS live in `sale_lines`; physical stock changes live in `stock_movements`.
**Why:** a made-to-order item has revenue but no stock of its own.
**Status:** Proposed

<a id="adr-006"></a>
### ADR-006 Weighted average cost (WAC)
**Decision:** each stocked item has a perpetual weighted average cost. Issues are booked at WAC, receipts at their actual cost, and production output at the cost of the components consumed.
**Why:** simple, stable, standard for food service, and needs no lot tracking.
**Alternatives rejected:** FIFO (needs lots; maybe in v2 with expiry dates); standard cost (needs manual upkeep).
**Status:** Proposed (Q-3)

<a id="adr-007"></a>
### ADR-007 Stocked vs made-to-order composites
**Decision:** a composite item is either Stocked (produce first, then sell from stock) or MadeToOrder (selling consumes its components, recursively). Bundles such as Tea + Cake are normally MadeToOrder.
**Why:** this answers the v0.1 questions "is a bundle physical or virtual?" and "deduct at creation or at sale?" per item, instead of with one global rule.
**Status:** Proposed (Q-1)

<a id="adr-008"></a>
### ADR-008 Reject negative stock by default
**Decision:** an operation that would make any balance negative is rejected as a whole, listing every shortage. The `allow_negative_stock` setting can turn this off.
**Why:** keeps the data honest. The setting exists for busy shops that record purchases late.
**Status:** Proposed (Q-2)

<a id="adr-009"></a>
### ADR-009 Pessimistic locks for stock, optimistic for catalog
**Decision:** stock operations use `SELECT … FOR UPDATE` on balances in item-id order. Catalog edits use the `xmin` token and return 409 on conflict.
**Why:** stock contention needs serialisation without retries; catalog edits are rare and a reload is acceptable.
**Status:** Proposed

<a id="adr-010"></a>
### ADR-010 Archive instead of delete once an item has history
**Decision:** hard delete only for items with no history; otherwise archive. The database enforces this with FK RESTRICT.
**Status:** Proposed

<a id="adr-011"></a>
### ADR-011 Margin-based pricing
**Decision:** suggested price = cost ÷ (1 − margin), rounded up to a configurable step.
**Why:** owners think in "what share of the price is profit". Markup is a common source of under-pricing.
**Status:** Proposed

<a id="adr-012"></a>
### ADR-012 Numeric precision and UUID v7
**Decision:** see [data model §3.5](03-data-model.md#35-precision-choices). Primary keys are UUID v7.
**Status:** Proposed

<a id="adr-013"></a>
### ADR-013 PostgreSQL
**Decision:** PostgreSQL 18 with snake_case naming.
**Why:** partial unique indexes, generated columns, check constraints, `xmin` concurrency, `numeric` type, free.
**Status:** Proposed

<a id="adr-014"></a>
### ADR-014 Report periods in the business time zone, configurable week start and calendar
**Decision:** store UTC; group by `business_settings.time_zone`; the week start and calendar (Gregorian or Jalali, via a `calendar_days` table) are settings.
**Status:** Proposed (Q-6)

<a id="adr-015"></a>
### ADR-015 Copy `parent_item_id` into recipe lines
**Decision:** recipe lines store the parent item, pinned to the recipe by a composite FK.
**Why:** it lets a database check constraint block self-composition, and it matches v0.1's parent → component shape.
**Status:** Proposed

<a id="adr-016"></a>
### ADR-016 ASP.NET Core Web API with controllers
**Decision:** the backend is an ASP.NET Core Web API (.NET 10, C#) that uses controllers (`[ApiController]`) and exposes REST + JSON, documented with OpenAPI and Swagger UI. It serves no pages: any frontend (web admin, tablet sales screen) is a separate client that calls this API.
**Why:** this is the owner's choice. Controllers give one class per module (`ItemsController`, `ProductionsController`…), built-in model validation, ProblemDetails, and per-endpoint role policies.
**Alternatives rejected:** Minimal APIs (less structure for about 50 endpoints); MVC or Razor Pages (would tie the UI to the server).
**Status:** **Accepted** (owner, 2026-11-15)

<a id="adr-017"></a>
### ADR-017 Subscription plans and a cheap default AI model
**Decision:** Chaayo is also sold to other cafés as a subscription, in three plans:
- Basic: 790,000 Toman a month, 2 users, no assistant.
- Pro: 1.69 million Toman a month, 6 users, 300 assistant commands.
- Smart: 3.49 million Toman a month, unlimited users, 3,000 assistant commands.

A yearly subscription is 20% off. Pro has a 14-day free trial. Optional setup costs 2.5 million Toman once. Extra assistant commands cost 190,000 Toman per 500. Routine assistant commands run on Claude Haiku 5.5. Analysis questions go to a stronger model, with their own quota.
**Why:** the prices sit against Iranian café software checked in November 2026: Biameno, Chashni, Abriron, Drop, Parmis and Hesabesh, which charge 0.4–6.4 million Toman a month. None of them has an AI assistant. On Claude Opus 5.5 alone, a Smart plan's 3,000 commands would cost about $120 a month, more than the plan's price. On Haiku 5.5 they cost about $6.
**Lifetime license:** a café can instead buy a lifetime license for 89 million Toman, once. It covers all Smart features for one café and includes setup and the first year of updates and support. From year two, support is optional at 15% of the current license price a year. Hosting is free on the café's own machine, or 490,000 Toman a month on our cloud. AI usage is billed monthly at actual cost, either through the buyer's own API key or as prepaid Toman credit at the day's exchange rate plus 20%. The license price is reviewed every 3 months for inflation. 89 million is about 2.7 years of the Smart plan billed yearly.
**Consequences:** the system needs per-café usage metering (assistant commands, users) and a model router in the AI adapter. Multi-tenant hosting (one deployment, many cafés) must be decided before selling subscriptions; the v0.2 design assumes one café per deployment.
**Status:** **Accepted** (owner, 2026-11-15)
