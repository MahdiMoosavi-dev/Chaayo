# 7. API Design

REST + JSON, base path `/api/v1`, documented live at `/swagger`. Auth: `Authorization: Bearer <JWT>`. Roles: **O** Owner, **S** Supervisor, **T** Staff.

All write endpoints accept an `Idempotency-Key: <uuid>` header (see [architecture §2.8](02-architecture.md#28-cross-cutting-concerns)). Lists support `?page=&pageSize=&sort=` and return `{ items, page, pageSize, total }`.

## 7.1 Endpoint catalogue

### Catalog

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/items` | Search: `q`, `type`, `categoryId`, `sellable`, `archived`, `lowStock` | O S T |
| POST | `/items` | Create a simple or composite item (composite with its first recipe) | O S |
| GET | `/items/{id}` | Details + balance + active recipe + cost + suggested price | O S T |
| PUT | `/items/{id}` | Edit details (needs `If-Match: <version>`) | O S |
| PUT | `/items/{id}/sales` | Set sellable, sale price, target margin | O |
| DELETE | `/items/{id}` | Hard delete; 422 `item.has_history` if it has history | O |
| POST | `/items/{id}/archive` · `/restore` | Archive or restore | O S |
| GET | `/items/{id}/price-history` | Price changes | O S |
| GET | `/items/{id}/where-used` | Recipes that use this item | O S |
| GET | `/items/{id}/cost` | Rolled-up cost, breakdown tree, suggested price, actual margin | O S |
| GET | `/items/{id}/capacity` | "What can I make?" + bottleneck | O S |
| GET | `/items/{id}/ledger` | Movements: `from`, `to` | O S |
| GET/POST/PUT/DELETE | `/categories` | Categories | O S (read: all) |
| GET | `/units` | Measurement units | all |

### Recipes

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/items/{id}/recipes` | All versions | O S |
| POST | `/items/{id}/recipes` | New version (`activate: true` by default) = "edit recipe" | O S |
| GET | `/recipes/{id}` | One version with lines | O S |
| POST | `/recipes/{id}/activate` | Switch the active version (runs the cycle check) | O S |
| GET | `/items/{id}/recipe-tree` | Fully expanded tree with quantities and costs | O S |

### Inventory and operations

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/inventory` | Balances, value, low-stock flags, totals | O S T |
| POST | `/purchases` | Record a purchase | O S |
| POST | `/productions/preview` | Requirements + shortages, no changes | O S |
| POST | `/productions` | Produce a composite item | O S |
| POST | `/sales` | Record a sale | O S T |
| POST | `/waste` | Record waste with a reason | O S T |
| POST | `/adjustments` | Manual ± with a reason (including opening balance) | O S |
| POST | `/stock-counts` | Counted quantities → variance movements | O S |
| GET | `/operations` | Filter by `type`, `from`, `to`, `itemId`, `userId` | O S |
| GET | `/operations/{id}` | Header + movements + sale lines | O S |
| POST | `/operations/{id}/reverse` | Reverse (reason required) | O S |
| POST | `/operations/{id}/correct` | Reverse + post the corrected operation | O S |

### Purchasing, reports, administration

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET/POST/PUT | `/suppliers`, `/suppliers/{id}` (+ `/archive`) | Suppliers | O S |
| GET | `/reports/summary` | Dashboard KPIs for `period=day\|week\|month\|custom`, `from`, `to`, with the previous-period comparison | O S |
| GET | `/reports/profit-and-loss` | Per day, week or month: revenue, COGS, gross profit, margin, spend, waste | O S |
| GET | `/reports/sales-by-item` | qty, revenue, COGS, profit per item | O S |
| GET | `/reports/purchases` | Spend by item, category, supplier | O S |
| GET | `/reports/waste` | Waste by reason and item | O S |
| GET | `/reports/inventory-value` | Value at each period end | O S |
| POST | `/auth/login` | Returns a JWT | — |
| GET/POST/PUT | `/users` | Manage users | O |
| GET/PUT | `/settings` | Business settings | O (read: all) |

## 7.2 Examples

**Produce 2 cakes**

```http
POST /api/v1/productions
Idempotency-Key: 0193a1f2-7c4e-7b1a-9f0e-2d1c3b4a5e6f

{ "itemId": "…cake…", "quantity": 2, "occurredAt": "2026-11-15T08:30:00+03:30", "note": "morning batch" }
```

```http
201 Created
{
  "operationId": "…", "number": 1042, "type": "Production", "recipeVersion": 1,
  "movements": [
    { "item": "Flour",  "quantityDelta": -1.0, "unitCost": 42000, "balanceAfter": 9.0 },
    { "item": "Sugar",  "quantityDelta": -0.6, "unitCost": 30000, "balanceAfter": 4.4 },
    { "item": "Egg",    "quantityDelta": -8,   "unitCost": 5000,  "balanceAfter": 22 },
    { "item": "Butter", "quantityDelta": -0.4, "unitCost": 200000,"balanceAfter": 1.6 },
    { "item": "Cake",   "quantityDelta": 2,    "unitCost": 90000, "balanceAfter": 2 }
  ]
}
```

**Shortage (nothing is changed)**

```http
422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://chaayo/errors/stock.insufficient",
  "title": "Not enough stock",
  "status": 422,
  "code": "stock.insufficient",
  "shortages": [
    { "itemId": "…", "item": "Egg", "required": 8, "available": 3, "unit": "pc" }
  ]
}
```

**Sale with a made-to-order item**

```http
POST /api/v1/sales
{ "lines": [
    { "itemId": "…tea-plus-cake…", "quantity": 1 },
    { "itemId": "…piece-of-cake…", "quantity": 2, "discount": 0 }
] }
```

`unitPrice` is optional on each line. When it's omitted, the item's current sale price is used and stored on the line.
