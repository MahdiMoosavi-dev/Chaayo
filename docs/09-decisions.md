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
