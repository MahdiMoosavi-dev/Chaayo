# 6. Reporting and Statistics

All statistics are **calculated from the ledger** (`stock_movements` and `sale_lines`). There are no separate counters that could drift. Because the ledger is append-only and every row has an `occurred_at`, any past period can be reproduced exactly.

## 6.1 KPI definitions

| KPI | Question it answers | Formula | Source |
|---|---|---|---|
| **Spend on raw items** | How much did we buy? | Σ `total_cost` of Purchase movements (minus reversed purchases) | `stock_movements` |
| **Revenue** | How much did we sell? | Σ `net_amount` | `sale_lines` |
| **Discounts given** | How much did we give away? | Σ `discount` | `sale_lines` |
| **COGS** | What did the sold items cost us? | Σ `cost_amount` | `sale_lines` |
| **Gross profit** | What did we earn on sales? | Revenue − COGS | |
| **Gross margin %** | How profitable are sales? | Gross profit ÷ Revenue | |
| **Waste cost** | What did we throw away? | −Σ `total_cost` of Waste movements | `stock_movements` |
| **Count variance** | What went missing (or was found)? | Σ `total_cost` of StockCount movements (negative = shrinkage) | `stock_movements` |
| **Operating profit (before expenses)** | What is left after losses? | Gross profit − Waste cost + Count variance | |
| **Inventory value** | What is on the shelves worth? | Σ `total_cost` of all movements up to the end of the period (under perpetual WAC this equals Σ quantity × average cost) | `stock_movements` |
| **Number of sales**, **average ticket** | How busy were we? | count of Sale operations; Revenue ÷ count | `inventory_operations` |
| **Top and bottom items** | What sells, what earns? | rank by quantity, revenue or gross profit | `sale_lines` |

### Spend vs profit: why there are two views

Buying 50 kg of flour on Monday is a big **spend** on Monday, but it is not a Monday **loss**: the flour is still on the shelf as inventory value. Profit counts only the cost of what was actually **sold** (COGS). The report shows both:

- **Cash view**: Spend vs Revenue. Money out vs money in.
- **Profit view**: Revenue − COGS − Waste. What the business really earned.

Over a long period the two converge: Spend = COGS + Waste + change in inventory value.

## 6.2 Reports

| Report | Grouping | Filters |
|---|---|---|
| **Summary dashboard** | Period total + trend by day, week or month + change vs the previous period | period |
| **Profit and loss** | Revenue, COGS, gross profit, margin, waste, variance | period, category |
| **Sales by item** | qty, revenue, COGS, profit, margin per item | period, category, item |
| **Purchases** | spend by item, category, supplier | period, supplier |
| **Waste** | cost by reason, by item | period, reason |
| **Inventory** | quantity, average cost, value, low-stock flag, days since last movement | category, low stock only |
| **Inventory value over time** | value at the end of each day, week or month | period |
| **Menu engineering** (later) | items in 4 quadrants: popularity × margin | period, category |

## 6.3 Dashboard sketch

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Today ▾   Week   Month   Custom          Compared with: previous period │
├──────────────┬──────────────┬──────────────┬──────────────┬──────────────┤
│ Revenue      │ COGS         │ Gross profit │ Margin       │ Spend        │
│ 4,280,000    │ 1,850,000    │ 2,430,000    │ 56.8 %       │ 3,100,000    │
│ ▲ 12 %       │ ▲ 9 %        │ ▲ 14 %       │ ▲ 1.1 pt     │ ▼ 4 %        │
├──────────────┴──────────────┴──────────────┴──────────────┴──────────────┤
│  Revenue / COGS / profit per day    ▁▃▅▇▆▅▇█▆▅▇                           │
├───────────────────────────────────┬──────────────────────────────────────┤
│ Top sellers (profit)              │ Alerts                               │
│ 1 Latte          612,000          │ ⚠ Butter low: 0.4 kg (reorder 1 kg) │
│ 2 Piece of Cake  355,000          │ ⚠ Piece of Cake margin 56% < 60%    │
│ 3 Tea + Cake     177,500          │ ⚠ Milk expires tomorrow (later)     │
├───────────────────────────────────┴──────────────────────────────────────┤
│ Waste today: 85,000 (Spilled 60,000 · Expired 25,000)                    │
└──────────────────────────────────────────────────────────────────────────┘
```

## 6.4 Example queries (PostgreSQL)

**Profit and loss per day** in the business time zone:

```sql
WITH sales AS (
    SELECT date_trunc('day', occurred_at AT TIME ZONE :tz) AS day,
           SUM(net_amount)  AS revenue,
           SUM(cost_amount) AS cogs
    FROM sale_lines
    WHERE occurred_at >= :from AND occurred_at < :to
    GROUP BY 1
), stock AS (
    SELECT date_trunc('day', m.occurred_at AT TIME ZONE :tz) AS day,
           SUM(m.total_cost) FILTER (WHERE o.type = 1)  AS spend,
           -SUM(m.total_cost) FILTER (WHERE o.type = 4) AS waste
    FROM stock_movements m
    JOIN inventory_operations o ON o.id = m.operation_id
    WHERE m.occurred_at >= :from AND m.occurred_at < :to
    GROUP BY 1
)
SELECT COALESCE(s.day, k.day) AS day,
       COALESCE(revenue, 0) AS revenue,
       COALESCE(cogs, 0) AS cogs,
       COALESCE(revenue, 0) - COALESCE(cogs, 0) AS gross_profit,
       COALESCE(spend, 0) AS spend,
       COALESCE(waste, 0) AS waste
FROM sales s FULL JOIN stock k ON k.day = s.day
ORDER BY 1;
```

Reversals are handled by sign. Reversed sales add negative sale lines, and reversed purchases add negative movements under a Reversal operation. The purchase report joins `reverses_operation_id` to find the original type, so a reversed purchase reduces spend **on the day of the reversal**. A closed day's report never changes later.

**Current inventory with low-stock flag:**

```sql
SELECT i.id, i.name, u.code AS unit, b.quantity, b.average_unit_cost,
       b.quantity * b.average_unit_cost AS value,
       (i.reorder_level IS NOT NULL AND b.quantity <= i.reorder_level) AS is_low
FROM items i
JOIN measurement_units u ON u.id = i.unit_id
LEFT JOIN stock_balances b ON b.item_id = i.id
WHERE NOT i.is_archived AND i.stock_mode = 1
ORDER BY is_low DESC, i.name;
```

## 6.5 Periods, time zone and calendar

- Timestamps are stored in **UTC** and grouped in the **business time zone** from `business_settings` (proposed: `Asia/Tehran`).
- **Day** = local midnight to midnight.
- **Week** starts on the configured day (proposed: Saturday). In SQL, shift the date before `date_trunc('week')`, which always starts on Monday: for a Saturday start, `date_trunc('week', local + interval '2 days') - interval '2 days'`.
- **Month**: needs your decision (Q-6).
  - **Gregorian** months work directly with `date_trunc('month')`.
  - **Jalali (Persian) months** don't line up with Gregorian ones. The proposed approach is a small `calendar_days` table (gregorian_date, jalali_year, jalali_month, jalali_day, week_start_date), generated once by the application using .NET's `PersianCalendar` for, say, 2020–2040. Reports join to it and group by `jalali_year, jalali_month`. This also gives correct Jalali week and month labels in the UI.

## 6.6 Performance plan

| Stage | Approach |
|---|---|
| MVP | Live queries on `sale_lines` and `stock_movements` with `(occurred_at)` and `(item_id, occurred_at)` indexes. Fine up to millions of rows for a single cafeteria. |
| If reports slow down | A nightly job (and on demand for today) fills `daily_item_summaries`. Reports read the summary for closed days and the live tables for today. |
