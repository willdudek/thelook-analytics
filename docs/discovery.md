# Discovery: thelook-analytics v1

Stage 2 (Discover) of the Two Brains loop. Completed 2026-10-15.
Source: TheLook eCommerce Kaggle snapshot in `data/raw/`. Profiled in the DuckDB v1.5.5 CLI; queries in Appendix A.
Numbering here (F-, C-, K-) supersedes the working numbers used in chat.

## 1. Sources

| Table | File | Rows | Grain | Key | In v1 |
|---|---|---:|---|---|---|
| orders | orders.csv | 124,923 | one row per order | order_id | Yes |
| order_items | order_items.csv | 180,952 | one row per order line | id | Yes |
| users | users.csv | 100,000 | one row per user | id | Yes |
| products | products.csv | 29,120 | one row per product | id | Yes (category) |
| inventory_items | inventory_items.csv | not profiled | | | No |
| distribution_centers | distribution_centers.csv | not profiled | | | No |
| events | events.csv | not profiled | | | No (funnel is v1.1) |

## 2. Schema notes (v1 tables)

- **orders:** order_id, user_id, status, gender, created_at, returned_at, shipped_at, delivered_at, num_of_item. No amount column.
- **order_items:** id, order_id, user_id, product_id, inventory_item_id, status, created_at, shipped_at, delivered_at, returned_at, sale_price (DOUBLE).
- **users:** id, first_name, last_name, email, age, gender, state, street_address, postal_code, city, country, latitude, longitude, traffic_source, created_at.
- **products:** id, cost (DOUBLE), category, name, brand, retail_price (DOUBLE), department, sku, distribution_center_id.

All columns infer as nullable (CSV inference); actual null checks are in the findings.

## 3. Findings

### Keys and joins
- **F-1.** All four tables have unique, non-null keys.
- **F-2.** No orphans: every item has an order and a product; every order has a user. Joins from items up to orders, users, and products are many-to-one, so they do not fan out.
- **F-3.** `orders.num_of_item` equals the count of `order_items` rows for all 124,923 orders.
- **F-4.** `order_items.user_id` differs from `orders.user_id` for 4 items, all on order_id 56 (items 88-91: item user 44, order user 56). This is the only discrepancy.

### Status and lifecycle
- **F-5.** Five statuses at both levels: Processing, Shipped, Complete, Cancelled, Returned. Order counts: Shipped 37,654; Complete 31,028; Processing 24,948; Cancelled 18,723; Returned 12,570.
- **F-6.** Item status always equals its order's status (all 180,952 items). There are no partial cancellations or partial returns.
- **F-7.** The status mix is roughly constant by year, including about 20% Processing among 2019 orders. Status is a generated attribute, not a lifecycle that resolves over time.
- **F-8.** Lifecycle timestamps match status exactly: Processing and Cancelled have none; Shipped has shipped_at only; Complete has shipped_at and delivered_at; Returned has all three. There is no cancelled_at column.
- **F-9.** Timestamp order is clean: nothing shipped before created, delivered before shipped, or returned before delivered.

### Amounts
- **F-10.** `sale_price` has no nulls and no values at or below zero. Range is about 0.02 to 999.00. Sum across all statuses: 10,758,880.13.
- **F-11.** Money is stored as DOUBLE with float artifacts (minimum `0.0199999995529651`).
- **F-12.** There are no tax, shipping, discount, or currency columns.
- **F-13.** `products` has `cost` and `retail_price`. Cost is per product, not per sale date.

### Time
- **F-14.** `orders.created_at` runs from 2019-01-12 to 2022-06-08 (a Wednesday). `users.created_at` runs from 2019-01-02 to 2022-06-07. `returned_at` runs to 2022-06-17.
- **F-15.** Daily orders grow gradually (about 370-470 per day in mid-to-late May), then spike June 3-6 (670, 917, 1,193, 1,869) and drop back to about 570-590 on June 7-8. The week of 2022-05-30 has 4,814 orders (vs 3,200 the week before), mostly from the spike. The week of 2022-06-06 is partial (3 days).
- **F-16.** `order_items.created_at` does not match the order's `created_at`: 102,442 items are earlier and 78,506 later, ranging from 3h58m before to 4 days after. Item created_at is not a reliable event time.
- **F-17.** No order is placed before its user's `created_at`.
- **F-18.** No timestamp carries timezone information.
- **F-20.** Return lag (orders.created_at to returned_at, all 12,570 returned orders): min 0 days, p50 5, p90 8, p99 10, max 11. No long tail.

### PII
- **F-19.** `users` holds first and last name, email, street address, postal code, latitude, and longitude. No v1 metric needs them.

### Anchor values
- **F-21.** Hand-verified, ISO week 2022-05-30 (#27, Q17): customers = 3601, new_customers = 2361, returning_customers = 1240. new + returning = customers.


## 4. Constraints

- **C-1.** The data is synthetic (F-6, F-7, F-15). Patterns are not evidence of real customer behavior; the project practices the workflow, not insight-finding.
- **C-2.** Static snapshot. No new data arrives, so "this week" is a fixed as-of date.
- **C-3.** Net revenue can only be sale price less cancellations and returns (F-12).
- **C-4.** Margin is possible only at static product cost (F-13). Out of v1 scope (decision 6).

## 5. Known unknowns

| # | Question | Why it matters | Resolve by | Resolved by |
|---|---|---|---|---|
| K-1 | Revenue definition: which statuses count, and which timestamp dates it | Status can't be read as "not yet shipped" (F-7), and there is no cancelled_at (F-8) | Decision in strategy/spec | Decision 10 (K1-B) |
| K-2 | As-of week: 2022-05-30 (last complete week, contains the F-15 spike) or 2022-05-23 (last week before the spike) | Sets the "current week" the founder sees | Decision in strategy | Decision 11 (K2-A) |
| K-3 | Return lag: how long after an order a return can land | Sets how many recent weeks are provisional | Query in spec stage | Decision 24 (option 1; F-20) |
| K-4 | Timezone of timestamps | ISO week boundaries depend on it | Decision (proposed: assume UTC, log it) | Decision 12 (K4-A) |
| K-5 | Order 56 user attribution (F-4) | Source of truth for user on items | Decision (proposed: orders.user_id) | Decision 13 (K5-A) |
| K-6 | Customer definitions: active, new vs returning | Core v1 metrics | Spec | Decision 25 (option 1) |
| K-7 | Event time for item-level metrics | Item created_at is unreliable (F-16) | Spec (proposed: orders.created_at) | Decision 26 (option 1) |
| K-8 | Item-level shipped/delivered/returned timestamps vs order-level: not compared | Only matters if the spec uses item timestamps | Query if needed | Decision 26 (item timestamps dropped; comparison not needed) |

## 6. Carried to stage 4 (spec rule and test candidates)

1. Cast money columns to DECIMAL in staging (F-11).
2. Drop PII columns in staging (F-19).
3. Invariant test: num_of_item equals item count per order (F-3).
4. Invariant test: item status equals order status (F-6). Catches a change in the data.
5. Schema tests: unique and not_null keys, relationships between tables (F-1, F-2).
6. Reconciliation anchor: all-status sale_price total of 10,758,880.13 from raw should survive staging unchanged (F-10).
7. Source test: item/order user mismatch count equals 4 (F-4); warn if it changes (#13).

## Appendix A: Profiling queries

Run from the repo root in an in-memory `duckdb` session. Recreate the views each session.

```sql
.mode markdown
.maxrows 50

CREATE VIEW orders   AS SELECT * FROM read_csv_auto('data/raw/orders.csv');
CREATE VIEW items    AS SELECT * FROM read_csv_auto('data/raw/order_items.csv');
CREATE VIEW users    AS SELECT * FROM read_csv_auto('data/raw/users.csv');
CREATE VIEW products AS SELECT * FROM read_csv_auto('data/raw/products.csv');

-- Schema
DESCRIBE orders; DESCRIBE items; DESCRIBE users; DESCRIBE products;

-- Q1 key uniqueness and nulls (F-1)
SELECT 'orders' t, COUNT(*) n, COUNT(DISTINCT order_id) d, COUNT(*) - COUNT(order_id) null_ct FROM orders
UNION ALL SELECT 'items', COUNT(*), COUNT(DISTINCT id), COUNT(*) - COUNT(id) FROM items
UNION ALL SELECT 'users', COUNT(*), COUNT(DISTINCT id), COUNT(*) - COUNT(id) FROM users
UNION ALL SELECT 'products', COUNT(*), COUNT(DISTINCT id), COUNT(*) - COUNT(id) FROM products;

-- Q2 date ranges (F-14)
SELECT 'orders.created_at' c, MIN(created_at), MAX(created_at) FROM orders
UNION ALL SELECT 'orders.returned_at', MIN(returned_at), MAX(returned_at) FROM orders
UNION ALL SELECT 'items.created_at', MIN(created_at), MAX(created_at) FROM items
UNION ALL SELECT 'items.returned_at', MIN(returned_at), MAX(returned_at) FROM items
UNION ALL SELECT 'users.created_at', MIN(created_at), MAX(created_at) FROM users;

-- Q3 status values (F-5)
SELECT 'order' lvl, status, COUNT(*) FROM orders GROUP BY ALL
UNION ALL SELECT 'item', status, COUNT(*) FROM items GROUP BY ALL
ORDER BY 1, 3 DESC;

-- Q4 order vs item status (F-6)
SELECT o.status order_status, i.status item_status, COUNT(*)
FROM items i JOIN orders o USING (order_id)
GROUP BY ALL ORDER BY 1, 3 DESC;

-- Q5 num_of_item vs item count (F-3)
SELECT (o.num_of_item = c.n) matches, COUNT(*)
FROM orders o
LEFT JOIN (SELECT order_id, COUNT(*) n FROM items GROUP BY 1) c USING (order_id)
GROUP BY 1;

-- Q6 orphans and user consistency (F-2, F-4)
SELECT
  (SELECT COUNT(*) FROM items i LEFT JOIN orders o USING (order_id) WHERE o.order_id IS NULL) items_without_order,
  (SELECT COUNT(*) FROM items i LEFT JOIN products p ON i.product_id = p.id WHERE p.id IS NULL) items_without_product,
  (SELECT COUNT(*) FROM orders o LEFT JOIN users u ON o.user_id = u.id WHERE u.id IS NULL) orders_without_user,
  (SELECT COUNT(*) FROM items i JOIN orders o USING (order_id) WHERE i.user_id <> o.user_id) item_order_user_mismatch;

-- Q7 price sanity (F-10, F-11)
SELECT COUNT(*) - COUNT(sale_price) null_price,
       COUNT(*) FILTER (WHERE sale_price <= 0) nonpositive,
       MIN(sale_price), MAX(sale_price), ROUND(SUM(sale_price), 2) total
FROM items;

-- Q8 recent weekly volume (F-15)
SELECT date_trunc('week', created_at) wk, COUNT(*) orders
FROM orders GROUP BY 1 ORDER BY 1 DESC LIMIT 6;

-- Q9 daily orders, final three weeks (F-15)
SELECT created_at::DATE d, COUNT(*) orders FROM orders
WHERE created_at >= '2022-05-16' GROUP BY 1 ORDER BY 1;

-- Q10 status by year (F-7)
SELECT year(created_at) yr, status, COUNT(*) FROM orders GROUP BY ALL ORDER BY 1, 2;

-- Q11 lifecycle timestamps by status (F-8)
SELECT status, COUNT(*) n,
       COUNT(shipped_at) has_shipped, COUNT(delivered_at) has_delivered, COUNT(returned_at) has_returned
FROM orders GROUP BY 1 ORDER BY 1;

-- Q12 item vs order created_at (F-16)
SELECT COUNT(*) FILTER (WHERE i.created_at < o.created_at) item_before_order,
       COUNT(*) FILTER (WHERE i.created_at > o.created_at) item_after_order,
       MAX(i.created_at - o.created_at) max_lag, MIN(i.created_at - o.created_at) min_lag
FROM items i JOIN orders o USING (order_id);

-- Q13 timestamp ordering (F-9)
SELECT COUNT(*) FILTER (WHERE shipped_at < created_at) shipped_before_created,
       COUNT(*) FILTER (WHERE delivered_at < shipped_at) delivered_before_shipped,
       COUNT(*) FILTER (WHERE returned_at < delivered_at) returned_before_delivered
FROM orders;

-- Q14 orders before signup (F-17)
SELECT COUNT(*) orders_before_signup FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.created_at < u.created_at;

-- Q15 user mismatch detail (F-4)
SELECT i.id item_id, i.order_id, i.user_id item_user, o.user_id order_user, i.created_at, o.created_at
FROM items i JOIN orders o USING (order_id) WHERE i.user_id <> o.user_id;

-- Q16 return lag (F-20)
SELECT COUNT(*) returned_orders,
       MIN(date_diff('day', created_at, returned_at)) min_days,
       quantile_cont(date_diff('day', created_at, returned_at), 0.5)  p50,
       quantile_cont(date_diff('day', created_at, returned_at), 0.9)  p90,
       quantile_cont(date_diff('day', created_at, returned_at), 0.99) p99,
       MAX(date_diff('day', created_at, returned_at)) max_days
FROM orders WHERE returned_at IS NOT NULL;

-- Q17 hand-calculated query
.mode markdown
CREATE VIEW orders AS SELECT * FROM read_csv_auto('data/raw/orders.csv');

-- Q17 hand-verified number (F-21, #27)
WITH qualifying AS (            -- #25: every status except Cancelled
  SELECT user_id, created_at
  FROM orders
  WHERE status <> 'Cancelled'
),
first_order AS (                -- #25: earliest qualifying order over ALL history
  SELECT user_id, MIN(created_at) AS first_order_at
  FROM qualifying
  GROUP BY user_id
),
week_customers AS (             -- #11, #12, #26: ISO week of 2022-05-30, UTC, order time
  SELECT DISTINCT user_id       -- #13: customer = orders.user_id
  FROM qualifying
  WHERE created_at >= TIMESTAMP '2022-05-30'
    AND created_at <  TIMESTAMP '2022-06-06'
)
SELECT
  COUNT(*)                                                        AS customers,
  COUNT(*) FILTER (WHERE f.first_order_at >= TIMESTAMP '2022-05-30') AS new_customers,
  COUNT(*) FILTER (WHERE f.first_order_at <  TIMESTAMP '2022-05-30') AS returning_customers
FROM week_customers w
JOIN first_order f USING (user_id);
```
