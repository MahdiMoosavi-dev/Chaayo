# Inventory — System Design (v0.2)

| | |
|---|---|
| **Status** | Draft for review. No implementation starts until this is approved. |
| **Date** | 2026-11-15 |
| **Supersedes** | Extends [cafeteria_inventory_system_design.docx](../cafeteria_inventory_system_design.docx) (v0.1) |
| **Stack** | ASP.NET Core Web API with controllers (.NET 10, C#) · EF Core · PostgreSQL. See [ADR-016](09-decisions.md#adr-016). |

**Client proposal (phases, timeline, cost):** [Inventory-Proposal.pdf](Inventory-Proposal.pdf) (English) · [Inventory-Proposal.fa.pdf](Inventory-Proposal.fa.pdf) (فارسی) · [Inventory-Proposal.fa.html](Inventory-Proposal.fa.html) (Persian web page with effects; download and open in a browser)

## How to review (about 45 minutes)

Read in this order. Each document stands alone, and the diagrams render directly on GitHub.

| # | Document | What it answers | Review focus |
|---|---|---|---|
| 1 | [Requirements](01-requirements.md) · [فارسی](01-requirements.fa.md) | What the system must do, in clear terms | Does it match how the cafeteria really works? |
| 2 | [Data model](03-data-model.md) | **The data structure**: tables, columns, constraints, ERD | The most important review. Fixing it later is expensive. |
| 3 | [Domain model](04-domain-model.md) | Business objects, rules, lifecycles | Are the rules right? |
| 4 | [Business processes](05-business-processes.md) | Step-by-step flows, costing and pricing with worked examples | Walk the cake example and check the numbers. |
| 5 | [Reporting](06-reporting.md) | Daily, weekly and monthly statistics: definitions and formulas | Are these the numbers you want to see? |
| 6 | [Architecture](02-architecture.md) | How the software is structured | Technical review |
| 7 | [API](07-api.md) | Endpoints the frontend will call | Technical review |
| 8 | [Features and roadmap](08-features-roadmap.md) | Added features, split into MVP and later phases | Choose what goes in the MVP. |
| 9 | [Decision log](09-decisions.md) | Every design decision and the reason for it | Disagree with any? Comment on its ID. |

## Decisions that need your sign-off

These are the choices I made where your answer could change the design. Each links to its decision record.

| ID | Question | Proposed answer | Confirm? |
|---|---|---|---|
| Q-1 | How are bundles like "Tea + Cake" stocked? | Each composite item is either **Stocked** (produced ahead, holds stock) or **Made-to-order** (never stocked; selling it consumes its components). Tea + Cake would be made-to-order. See [ADR-007](09-decisions.md#adr-007). | ☐ |
| Q-2 | Can stock go negative? | **No** by default. A business setting can allow it for rush hours. See [ADR-008](09-decisions.md#adr-008). | ☐ |
| Q-3 | How is cost calculated? | **Weighted average cost** per item. See [ADR-006](09-decisions.md#adr-006). | ☐ |
| Q-4 | What happens when a recipe is edited? | A **new recipe version** is created and the old one is kept for history. See [ADR-003](09-decisions.md#adr-003). | ☐ |
| Q-5 | How are past mistakes fixed? | You never edit or delete history. A **reversal** cancels the operation and you re-enter it. See [ADR-004](09-decisions.md#adr-004). | ☐ |
| Q-6 | Calendar for monthly and weekly reports? | Configurable time zone and week start. **Jalali months?** See [Reporting §6.5](06-reporting.md#65-periods-time-zone-and-calendar). | ☐ needs your answer |
| Q-7 | Currency? | One currency, set in settings (for example IRR or Toman). | ☐ needs your answer |
| Q-8 | Frontend? | Out of scope for this design. Choose between React and Blazor before the UI phase. | ☐ later |
| Q-9 | MVP scope | The features marked **MVP** in [Features and roadmap](08-features-roadmap.md). | ☐ |

## What changed from v0.1

| v0.1 | v0.2 | Why |
|---|---|---|
| `ItemComposition (parent, component, qty)` | `Recipe` (versioned, with a yield) + `RecipeLine` (parent, component, qty) | Keeps history when recipes change, and supports "1 batch makes 24 cookies". The parent → component link from v0.1 survives in `RecipeLine`. |
| A shared `OperationId` on movements | `InventoryOperation` header table + `StockMovement` lines | Each event gets a type, a time, a user, a supplier and a reversal link in one place. |
| Not covered | `SaleLine`, `Supplier`, prices, costs, reports | Needed for your new requirements: pricing, suggested price, spend, sales and profit statistics. |
| Bundle semantics left open | Stocked vs Made-to-order | Resolves the open question in v0.1 §7. |
| "Delete item" | Hard delete only for items with no history; otherwise archive | Reports must never lose data. |

## Glossary

The glossary lives in [Requirements §3](01-requirements.md#3-glossary).
