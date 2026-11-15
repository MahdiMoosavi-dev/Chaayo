# 1. Requirements

> Persian version: [01-requirements.fa.md](01-requirements.fa.md)

## 1. Vision

Chaayo tells a cafeteria, at any moment:

- **what it has in storage**, and what that stock is worth;
- **what each product really costs to make**, down to the last gram of sugar;
- **what to charge** for each product to hit a target profit;
- **how much money it spends, earns and keeps**, by day, week and month.

## 2. The problem, restated

Your original description, rewritten as precise statements:

1. **Everything is an item.** Flour, a cake, a piece of cake, a bottle of tea and a "Tea + Cake" deal are all items in one catalog.
2. **Two kinds of items.**
   - A **simple item** is bought or counted as-is: flour, sugar, bottled tea. (This is the "custom" item in your description.)
   - A **composite item** is defined by a **recipe**: a list of other items and quantities. Components can themselves be composite, so a Tea + Cake deal contains a piece of cake, which comes from a cake, which is made of flour.
3. **Increasing a composite item consumes its components.** Making 2 cakes subtracts 2 × (the recipe's flour, sugar, eggs…) from stock and adds 2 cakes. All of these changes succeed together or not at all.
4. **Increasing a simple item** is a **purchase**, which records the price paid and the supplier, or a manual adjustment.
5. **Decreasing an item** is a **sale**, **waste** (with a reason) or a **manual adjustment**.
6. **Every item has a price.**
   - A **sale price**, for items that are sold.
   - A **cost**, calculated automatically from what was paid for it, or, for composite items, from the cost of its components.
7. **Composite items get a suggested price**: their component cost plus your target profit margin.
8. **Items can be created, edited, archived and deleted.**
   - Deleting is allowed only while an item has no history.
   - Once it has history it is archived (hidden), so reports stay correct.
9. **An inventory screen** shows each item's quantity, cost and value, low-stock warnings, and the full history of every change.
10. **Statistics, by day, week, month or a custom range:**
    - money spent on raw items (purchases)
    - sales revenue
    - cost of what was sold
    - gross profit and margin
    - waste
    - stock value
    - best and worst sellers

## 3. Glossary

| Term | Meaning |
|---|---|
| **Item** | Anything the cafeteria stocks or sells. |
| **Simple item** | Item with no recipe. It is bought or counted as-is. |
| **Composite item** | Item made from other items through a recipe. |
| **Recipe** | Versioned list of components and quantities for one composite item, plus a yield. |
| **Component** | An item used in a recipe. It can itself be composite. |
| **Yield** | How many units one batch of a recipe produces (1 cake, 24 cookies, 8 slices). |
| **Stocked** | A composite item that is produced ahead of time and held in stock (cakes). |
| **Made-to-order (MTO)** | A composite item never held in stock. Selling it consumes its components directly (a latte, Tea + Cake). |
| **Operation** | One business event that changes stock: purchase, production, sale, waste, adjustment, stock count or reversal. |
| **Movement** | One line of an operation: item X went up or down by N. The ledger only ever has movements added, never edited. |
| **Balance** | Current quantity on hand of an item. It always equals the sum of that item's movements. |
| **WAC** | Weighted average cost: the average price paid per unit for current stock. |
| **COGS** | Cost of goods sold: what the items you sold cost you. |
| **Gross profit** | Revenue minus COGS. |
| **Margin %** | Gross profit ÷ revenue. A 60% margin means 60 of every 100 you charge is profit. |
| **Markup %** | Gross profit ÷ cost. This is not the same as margin, and the system uses **margin**. |
| **Suggested price** | Cost ÷ (1 − target margin), rounded up to a round number. |
| **Reorder level** | When the balance falls to this level or below, the item shows as low stock. |
| **Archive** | Hide an item from daily use while keeping its history. |
| **Reversal** | An operation that cancels an earlier one with opposite movements. |

## 4. Users and roles

| Role | Typical person | Can do |
|---|---|---|
| **Owner** | Business owner | Everything: settings, users, prices, all reports, reversals |
| **Supervisor** | Shift manager | Items, recipes, purchases, production, waste, stock counts, reports |
| **Staff** | Cashier, barista | Record sales, record waste, view inventory |
| **System** | Background job | Low-stock alerts, daily summary |

## 5. Functional requirements

Priority: **MVP** = first release · **Later** = see the [roadmap](08-features-roadmap.md).

### 5.1 Catalog (items, recipes, categories, units)

| ID | Requirement | Priority |
|---|---|---|
| FR-CAT-01 | Create a **simple item**: name, unit, optional category, SKU, description, reorder level, sale price if sellable. | MVP |
| FR-CAT-02 | Create a **composite item** with its first recipe: components, quantities, yield, stock mode (stocked or made-to-order). Components are always existing items and are never copied. | MVP |
| FR-CAT-03 | Edit item details (name, description, category, SKU, reorder level, prices). An item's unit can change only while it has no stock history. | MVP |
| FR-CAT-04 | Edit a recipe. This saves a **new version** and makes it the active one; old versions stay for history. | MVP |
| FR-CAT-05 | Reject recipes that are circular (A contains B contains A), use an item as its own component, have zero or negative quantities, or repeat a component. | MVP |
| FR-CAT-06 | **Delete** an item only when it has no movements, no sales and no recipe uses it. Otherwise **archive** it. Archived items can be restored. | MVP |
| FR-CAT-07 | Block archiving an item that an active recipe still uses, and list the recipes that use it ("where used"). | MVP |
| FR-CAT-08 | Manage categories (Ingredients, Hot drinks, Pastries…) and search or filter items by name, SKU, category, type, sellable or archived. | MVP |
| FR-CAT-09 | Show the full recipe tree of an item (nested) with the quantity and cost of each component. | MVP |
| FR-CAT-10 | Units: g, kg, ml, l, piece, bottle, cup, pack, slice. Recipe quantities can be typed in another unit of the same kind (for example 500 g for an item stocked in kg) and are converted. | MVP |

### 5.2 Stock (increase, decrease, inventory)

| ID | Requirement | Priority |
|---|---|---|
| FR-STK-01 | **Purchase** (increases simple items): items, quantities, unit price paid, optional supplier and invoice number. | MVP |
| FR-STK-02 | **Production** (increases stocked composite items): choose the item and quantity. The system deducts components according to the active recipe and adds the product, in **one atomic transaction**. | MVP |
| FR-STK-03 | Before producing, show a **preview** of the required components and any shortages. | MVP |
| FR-STK-04 | If any component is short, **reject** the whole operation and list every shortage. (A setting can allow negative stock instead, see ADR-008.) | MVP |
| FR-STK-05 | **Sale**: one or more lines with item, quantity, price (defaults to the item's sale price) and optional discount. Stocked items deduct their own stock. Made-to-order items deduct their components, recursively for nested made-to-order components. | MVP |
| FR-STK-06 | **Waste**: decrease with a reason (expired, damaged, spilled, staff meal…). | MVP |
| FR-STK-07 | **Adjustment**: manual increase or decrease with a reason, for example the opening balance. | MVP |
| FR-STK-08 | **Stock count**: enter counted quantities. The system records the variance (counted minus expected) as movements. | MVP |
| FR-STK-09 | **Reverse** any operation, which creates opposite movements. History is never edited or deleted. "Correcting" an operation means reversing it and re-entering it in one step. | MVP |
| FR-STK-10 | **Inventory screen**: per item, the quantity, unit, average cost, stock value, reorder level, low-stock flag and last movement date. Filters and totals included. | MVP |
| FR-STK-11 | **Item ledger**: every movement of an item with date, operation type, quantity, cost, balance after and user. | MVP |
| FR-STK-12 | **What can I make?** The maximum quantity of a composite item that current stock allows, and which component limits it. | MVP |
| FR-STK-13 | Low-stock alert list and a suggested shopping list. | Later |

### 5.3 Pricing and cost

| ID | Requirement | Priority |
|---|---|---|
| FR-PRC-01 | Each sellable item has a **sale price**, and a price is required before it can be sold. Each price change is recorded with who changed it and when. | MVP |
| FR-PRC-02 | Each stocked item has a **current unit cost**, the weighted average of what was paid, updated on every purchase or production. | MVP |
| FR-PRC-03 | Composite items show a **rolled-up cost**: the sum of component costs through the recipe tree, divided by the yield. | MVP |
| FR-PRC-04 | Composite items show a **suggested price** from the cost and the item's target margin (or the default margin), rounded up to a configurable step (for example 1,000). | MVP |
| FR-PRC-05 | Show each item's actual margin % at its current price, and warn when it falls below the target, for example after an ingredient gets more expensive. | MVP |
| FR-PRC-06 | Show a cost breakdown: which component contributes how much to the cost. | MVP |

### 5.4 Purchasing

| ID | Requirement | Priority |
|---|---|---|
| FR-PUR-01 | Manage suppliers (name, phone, notes). | MVP |
| FR-PUR-02 | Show the last price paid per item and per supplier. | Later |

### 5.5 Reporting

| ID | Requirement | Priority |
|---|---|---|
| FR-RPT-01 | Choose a period: today, a day, a week, a month or a custom range, in the business time zone. | MVP |
| FR-RPT-02 | **Spend on raw items**: total purchases in the period, by item, category and supplier. | MVP |
| FR-RPT-03 | **Sales**: revenue, quantity sold and discounts, by item and category. | MVP |
| FR-RPT-04 | **Profit**: revenue, COGS, gross profit and margin %, in total, per item and per category. | MVP |
| FR-RPT-05 | **Waste**: cost of waste by reason and by item. | MVP |
| FR-RPT-06 | **Stock value** now, and at the end of each period. | MVP |
| FR-RPT-07 | Top and bottom items by quantity, revenue and profit. | MVP |
| FR-RPT-08 | A trend chart of daily revenue, COGS and profit over the selected range, compared with the previous period. | MVP |
| FR-RPT-09 | Export any report to CSV or Excel. | Later |
| FR-RPT-10 | **Menu engineering**: classify sellable items by popularity × margin. | Later |

### 5.6 Administration

| ID | Requirement | Priority |
|---|---|---|
| FR-ADM-01 | Users with roles (Owner, Supervisor, Staff); log in and out. | MVP |
| FR-ADM-02 | Business settings: currency, time zone, week start day, default target margin, price rounding step, allow negative stock. | MVP |
| FR-ADM-03 | Every operation records who made it and when. | MVP |

## 6. Non-functional requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-01 | Correctness | All stock changes of one operation are committed in a single database transaction. A failure changes nothing. |
| NFR-02 | Concurrency | Two people selling the last piece of cake at the same moment cannot both succeed. |
| NFR-03 | Auditability | The movement ledger is append-only. The balance of each item always equals the sum of its movements, and a nightly check verifies this. |
| NFR-04 | Precision | Quantities and money use `decimal` and are never floating point. Quantities have 4 decimals and unit costs 6. |
| NFR-05 | Performance | The inventory list (5,000 items) returns in under 500 ms. A monthly report over 200,000 movements returns in under 2 s. Recording a sale takes under 300 ms. |
| NFR-06 | Security | HTTPS only. Hashed passwords. Role checks on every endpoint. Secrets are never stored in the repository. |
| NFR-07 | Reliability | Daily database backup, keeping 30 days. Write endpoints are idempotent, so a retry on bad Wi-Fi does not double-record a sale. |
| NFR-08 | Localisation | UI text can be translated (Persian and English), with RTL support in the frontend. Numbers are formatted per locale. |
| NFR-09 | Maintainability | Modular monolith with separate layers. Automated tests cover every business rule in this document. |

## 7. Acceptance scenarios (examples)

**S-1 Production consumes components**
Given flour 10 kg, sugar 5 kg, eggs 30 pc and butter 2 kg, and a cake recipe (yield 1) of flour 0.5 kg, sugar 0.3 kg, eggs 4 and butter 0.2 kg.
When a supervisor produces 2 cakes.
Then flour is 9 kg, sugar 4.4 kg, eggs 22 and butter 1.6 kg, and cake is 2. Five movements share one operation.

**S-2 Shortage rejects everything**
Given eggs at 3 pc.
When a supervisor produces 1 cake, which needs 4 eggs.
Then the operation is rejected with "Eggs: need 4, have 3". No balance changes.

**S-3 Made-to-order sale**
Given Tea + Cake (made-to-order) = 1 piece of cake + 1 bottle of tea, priced at 44,000.
When staff sell 1 Tea + Cake.
Then piece of cake goes down by 1 and bottle of tea by 1. Revenue is 44,000 and COGS is the cost of both components.

**S-4 Recipe edit keeps history**
Given 10 cakes were produced with recipe v1.
When the supervisor changes the flour to 0.6 kg.
Then recipe v2 is active, and the 10 earlier productions still show v1.

**S-5 Deleting an item with history**
Given flour has purchases.
When someone deletes flour.
Then the request is refused with "Flour has history; archive it instead".

## 8. Out of scope for the MVP

Multi-branch support, accounting and tax ledgers, payroll, online ordering, customer loyalty, hardware integrations (printers, scales). Some of these appear in the [roadmap](08-features-roadmap.md).
