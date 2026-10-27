# Weekly metrics (Spec 1) -- Spec v1.0.2

Stage 4 (Spec) of the Two Brains loop. Drafted 2026-10-27.
References: F-, C-, K- point to docs/discovery.md; # to docs/decisions.md; P-, M-, S-, I-, MT-, T-, U- are defined in this spec.
Companion spec: docs/specs/weekly-dashboard.md (Spec 1b, #23) builds the page from the mart defined here.

## 1. Purpose and scope

Purpose: one weekly mart, `mart_weekly_metrics`, that gives the founder's Monday review a single, tested set of numbers for sales, customers, and cancellations/returns, and explains week-over-week change (strategy brief, success 1-3).

In scope:
- Source definitions over the raw CSVs.
- Staging: `stg_orders`, `stg_order_items`, `stg_products`.
- Intermediate: `int_order_items`, `int_orders`.
- Mart: `mart_weekly_metrics`.
- Schema tests, singular tests, and dbt unit tests in section 7.

Out of scope (do not build):
1. The Evidence dashboard and anything in dashboard/ (Spec 1b, #23).
2. Retention, churn, 6-month active customers, cohort models (Spec 2, #16).
3. Category metrics and the funnel (v1.1, #15, #6). `category` is carried, not used.
4. Margin and product cost (C-4).
5. Inventory, distribution centers, events tables (#6).
6. A Python loader, scheduling, MotherDuck (#6, #18).
7. Percent change, orders per customer, return rate (#31).
8. Timezone conversion (#12).
9. Incremental models, snapshots, seeds, new dbt packages (#37).

## 2. Inputs

### 2.1 Sources

| Source table | File | Rows | Key | Used for |
|---|---|---:|---|---|
| orders | data/raw/orders.csv | 124,923 | order_id | stg_orders; T-3, T-4, T-15, T-19 |
| order_items | data/raw/order_items.csv | 180,952 | id | stg_order_items; T-3, T-4, T-15 |
| products | data/raw/products.csv | 29,120 | id | stg_products |
| users | data/raw/users.csv | 100,000 | id | relationship target only (T-2); no staging model (#34) |

Raw files are read in place and never modified or copied (#1). Column inventory: discovery.md section 2.

### 2.2 Arrival pattern

Static snapshot (C-2). No new data arrives; refresh is manual (8.4).

### 2.3 Sample data

data/samples/ (gitignored, #19). Not used by tests; tests run on full data (#18) and on unit-test fixtures.

### 2.4 Environment

- dbt-core 1.8 or later (dbt unit tests) and dbt-duckdb, managed by uv in pyproject.toml / uv.lock.
- dbt_project.yml at repo root holds materializations per layer and the `return_window_days` var (section 5).
- profiles.yml at repo root points to the local dev.duckdb; no secrets (#18).
- E-1 (answered 2026-10-27): pyproject.toml, uv.lock, dbt_project.yml, and profiles.yml do not exist. SETUP-01 creates them.
- E-2 (answered 2026-10-27): dbt is not installed. SETUP-01 pins dbt-core >= 1.8 and dbt-duckdb.

### 2.5 Page outline (input to the mart; Spec 1b owns the build)

Context:
- P-1. As-of week label and data-through date. Needs is_as_of_week, week_start, last_order_date.
- P-2. Synthetic-data caveat (C-1). Static text.
- P-3. Provisional caveat. Always shown for the as-of week (#33). Needs is_provisional.

As-of week:
- P-4. Revenue waterfall: gross - cancelled - returned = net (#10).
- P-5. Orders: qualifying orders (#29).
- P-6. Customers, new and returning (#25).
- P-7. AOV (#30).
- P-8. Change vs prior week for P-4 to P-7 (#31). Needs prior_ and change_ columns.

Trend:
- P-9. Weekly net revenue, optionally with gross, cancelled, returned.
- P-10. Weekly customers, new and returning.
- P-11. Weekly orders and AOV.
- P-12. Incomplete and provisional weeks marked. Needs is_complete, is_provisional. Display and trend window decided in Spec 1b.

Not on the page: category, funnel, 6-month active, churn, retention, margin, orders per customer, return rate.

## 3. Outputs

### 3.1 Lineage

```
sources (data/raw/*.csv)
  ├─ stg_orders ──────────────────────────┐
  ├─ stg_order_items ─┐                   │
  └─ stg_products ────┴─ int_order_items ─┴─ int_orders ── mart_weekly_metrics
```

`mart_weekly_metrics` reads `int_orders` only. Materialization: staging views; intermediate and mart tables (#14).

### 3.2 Metric table

Common grain: one value per ISO week, Monday start, UTC (#11, #12). Week assignment from orders.created_at (#26). Customer = orders.user_id (#13). Status = order status (F-6).

Week attributes:

| # | Name | Definition | Pseudocode | Cites |
|---|---|---|---|---|
| M-01 | week_start | Monday of the ISO week, UTC, as DATE | `date_trunc('week', orders.created_at)::DATE` | #11, #12, rule 4.2 |
| M-02 | last_order_date | Date of the latest order; same value on every row | `MAX(orders.created_at)::DATE` | #24, #28 |
| M-03 | is_complete | Week's Sunday on or before last_order_date; first week counts as complete | `week_start + 6 <= last_order_date` | #28 |
| M-04 | is_provisional | Week end plus return window is after last_order_date | `week_start + 6 + var('return_window_days') > last_order_date` | #24 |
| M-05 | is_as_of_week | Latest complete week; exactly one row true | `week_start = MAX(week_start) WHERE is_complete` | #11, #33 |

Metrics:

| # | Name | Definition (words) | Pseudocode | Sources | Caveats / cites |
|---|---|---|---|---|---|
| M-10 | gross_revenue | Sale price of all items, any status | `SUM(order_sale_price)` | int_orders | All weeks sum to 10,758,880.13 (F-10). #10 |
| M-11 | cancelled_revenue | Sale price on Cancelled orders; stored positive, subtracted in the waterfall | `SUM(order_sale_price) WHERE status = 'Cancelled'` | int_orders | Order-dated; no cancelled_at (F-8). #10 |
| M-12 | returned_revenue | Sale price on Returned orders; stored positive | `SUM(order_sale_price) WHERE status = 'Returned'` | int_orders | Order-dated; provisional weeks can still grow (#24). #10 |
| M-13 | net_revenue | Gross less cancelled less returned | `gross_revenue - cancelled_revenue - returned_revenue` | mart | Checked by an independent raw path, T-15 (#21). #10 |
| M-14 | orders | Qualifying orders: any status except Cancelled | `COUNT(DISTINCT order_id) WHERE is_qualifying` | int_orders | Includes Returned. #29 |
| M-15 | customers | Distinct customers with a qualifying order in the week | `COUNT(DISTINCT user_id) WHERE is_qualifying` | int_orders | Customers with only cancelled orders excluded. F-21: 3,601. #25 |
| M-16 | new_customers | Customers whose first qualifying order (all history) is in this week | `COUNT(DISTINCT user_id) WHERE is_qualifying AND date_trunc('week', first_qualifying_order_at)::DATE = week_start` | int_orders | F-21: 2,361. #25, #27 |
| M-17 | returning_customers | Customers in the week whose first qualifying order is in an earlier week | `COUNT(DISTINCT user_id) WHERE is_qualifying AND date_trunc('week', first_qualifying_order_at)::DATE < week_start` | int_orders | Computed by its own condition, never customers - new (rule 4.11). F-21: 1,240. #25, #35 |
| M-18 | aov | Qualifying revenue (gross less cancelled, includes returned orders) per qualifying order. Basket size before returns. | `ROUND((gross_revenue - cancelled_revenue) / orders, 2)`; null if orders = 0 | mart | Not net AOV; returns show in the waterfall. #30 |

Prior-week comparison:

| # | Name | Definition | Pseudocode | Cites |
|---|---|---|---|---|
| M-20 | prior_<m> | Metric value for the week starting 7 days earlier | join `prior.week_start = cur.week_start - 7` | #31 |
| M-21 | change_<m> | Metric minus prior; null if either is null | `<m> - prior_<m>` | #31 |

<m> is each of: gross_revenue, cancelled_revenue, returned_revenue, net_revenue, orders, customers, new_customers, returning_customers, aov. Both columns null in the first week.

### 3.3 Staging models

**S-1. stg_orders** (view). Grain: one row per order. Key: order_id.

| Column | Type | Nullable | Source / note |
|---|---|---|---|
| order_id | BIGINT | no | orders.order_id |
| user_id | BIGINT | no | orders.user_id; customer of record (#13) |
| status | VARCHAR | no | orders.status (F-5) |
| created_at | TIMESTAMP | no | orders.created_at; event time, UTC (#12, #26) |
| returned_at | TIMESTAMP | yes | orders.returned_at; return-lag test only (#24) |
| num_of_item | INTEGER | no | orders.num_of_item; item-count invariant only (F-3) |

Dropped: gender, shipped_at, delivered_at.

**S-2. stg_order_items** (view). Grain: one row per order line. Key: order_item_id.

| Column | Type | Nullable | Source / note |
|---|---|---|---|
| order_item_id | BIGINT | no | order_items.id, renamed (#34) |
| order_id | BIGINT | no | order_items.order_id |
| product_id | BIGINT | no | order_items.product_id |
| sale_price | DECIMAL(18,2) | no | order_items.sale_price, cast per row (rule 4.5, #32) |

Dropped: user_id (#13), status (F-6), created_at, shipped_at, delivered_at, returned_at (#26), inventory_item_id (#6).

**S-3. stg_products** (view). Grain: one row per product. Key: product_id.

| Column | Type | Nullable | Source / note |
|---|---|---|---|
| product_id | BIGINT | no | products.id, renamed (#34) |
| category | VARCHAR | yes | products.category; carried for v1.1, untested (#15, #36) |

Dropped: all other columns, including cost and retail_price (C-4).

### 3.4 Intermediate models

**I-1. int_order_items** (table). Grain: one row per order line. Key: order_item_id. stg_order_items joined to stg_orders (many-to-one) and left-joined to stg_products (rule 4.22).

| Column | Type | Nullable | Source / rule |
|---|---|---|---|
| order_item_id | BIGINT | no | stg_order_items |
| order_id | BIGINT | no | stg_order_items |
| product_id | BIGINT | no | stg_order_items |
| category | VARCHAR | yes | stg_products (#15) |
| sale_price | DECIMAL(18,2) | no | stg_order_items |
| user_id | BIGINT | no | stg_orders (4.3) |
| status | VARCHAR | no | stg_orders (4.4) |
| order_created_at | TIMESTAMP | no | stg_orders.created_at, renamed (4.1, #34) |
| week_start | DATE | no | M-01 on order_created_at |

**I-2. int_orders** (table). Grain: one row per order. Key: order_id. stg_orders plus int_order_items aggregated to order_id.

| Column | Type | Nullable | Source / rule |
|---|---|---|---|
| order_id | BIGINT | no | stg_orders |
| user_id | BIGINT | no | stg_orders (4.3) |
| status | VARCHAR | no | stg_orders |
| order_created_at | TIMESTAMP | no | stg_orders.created_at, renamed |
| week_start | DATE | no | M-01 |
| returned_at | TIMESTAMP | yes | stg_orders; T-11 only |
| num_of_item | INTEGER | no | stg_orders; T-9 only |
| item_count | INTEGER | no | count of int_order_items rows for the order |
| order_sale_price | DECIMAL(18,2) | no | sum of int_order_items.sale_price for the order |
| is_qualifying | BOOLEAN | no | status <> 'Cancelled' (4.9) |
| first_qualifying_order_at | TIMESTAMP | yes | earliest order_created_at among the user's qualifying orders, all history; same on every order of the user; null if none (4.10, #35) |

### 3.5 Mart

**MT-1. mart_weekly_metrics** (table). Grain: one row per ISO week from 2019-01-07 through 2022-06-06 on a spine (4.13). Key: week_start. This column list, in this order, is the interface Spec 1b builds against.

| # | Column | Type | Nullable | Metric |
|---|---|---|---|---|
| 1 | week_start | DATE | no | M-01 |
| 2 | last_order_date | DATE | no | M-02 |
| 3 | is_complete | BOOLEAN | no | M-03 |
| 4 | is_provisional | BOOLEAN | no | M-04 |
| 5 | is_as_of_week | BOOLEAN | no | M-05 |
| 6 | gross_revenue | DECIMAL(18,2) | no | M-10 |
| 7 | cancelled_revenue | DECIMAL(18,2) | no | M-11 |
| 8 | returned_revenue | DECIMAL(18,2) | no | M-12 |
| 9 | net_revenue | DECIMAL(18,2) | no | M-13 |
| 10 | orders | BIGINT | no | M-14 |
| 11 | customers | BIGINT | no | M-15 |
| 12 | new_customers | BIGINT | no | M-16 |
| 13 | returning_customers | BIGINT | no | M-17 |
| 14 | aov | DECIMAL(18,2) | yes | M-18 |
| 15-16 | prior_gross_revenue, change_gross_revenue | DECIMAL(18,2) | yes | M-20, M-21 |
| 17-18 | prior_cancelled_revenue, change_cancelled_revenue | DECIMAL(18,2) | yes | M-20, M-21 |
| 19-20 | prior_returned_revenue, change_returned_revenue | DECIMAL(18,2) | yes | M-20, M-21 |
| 21-22 | prior_net_revenue, change_net_revenue | DECIMAL(18,2) | yes | M-20, M-21 |
| 23-24 | prior_orders, change_orders | BIGINT | yes | M-20, M-21 |
| 25-26 | prior_customers, change_customers | BIGINT | yes | M-20, M-21 |
| 27-28 | prior_new_customers, change_new_customers | BIGINT | yes | M-20, M-21 |
| 29-30 | prior_returning_customers, change_returning_customers | BIGINT | yes | M-20, M-21 |
| 31-32 | prior_aov, change_aov | DECIMAL(18,2) | yes | M-20, M-21 |

## 4. Rules

Time and attribution
- 4.1 Every item and order takes its time from orders.created_at. No item timestamp is used anywhere (#26).
- 4.2 Week = ISO week, Monday start: `date_trunc('week', ts)` on the UTC timestamp, cast to DATE where computed. No timezone conversion; any future conversion goes in staging only (#11, #12).
- 4.3 Customer = orders.user_id. order_items.user_id is dropped in staging (#13).
- 4.4 Status = order status. order_items.status is dropped in staging (F-6).

Columns and money
- 4.5 sale_price is cast to DECIMAL(18,2) once, per row, in stg_order_items. All downstream sums use the decimal value. No other rounding except 4.12 (#32, F-11).
- 4.6 No users column appears in any model (F-19).
- 4.7 Columns dropped in staging are exactly those listed in S-1 to S-3 and are not reintroduced downstream.

Metric definitions
- 4.8 Revenue waterfall: M-10 to M-13 as written (#10).
- 4.9 Qualifying order = status <> 'Cancelled' (#25).
- 4.10 first_qualifying_order_at = earliest order_created_at among a user's qualifying orders, over all history; null if none (#25, #35).
- 4.11 orders, customers, new_customers, returning_customers per M-14 to M-17. New vs returning compares `date_trunc('week', first_qualifying_order_at)::DATE` with week_start. returning_customers is computed by its own condition; deriving it as customers - new_customers is not allowed (#25, #35).
- 4.12 aov = `ROUND((gross_revenue - cancelled_revenue) / orders, 2)` cast to DECIMAL(18,2); null when orders = 0 (#30).

Week flags and spine
- 4.13 Spine: one row per ISO week from the min to the max week_start in int_orders, generated with DuckDB `generate_series` (no packages). Weeks without orders get 0 for sums and counts, not null (#33).
- 4.14 last_order_date = `MAX(orders.created_at)::DATE`, same value on every row (#24, #28).
- 4.15 is_complete = `week_start + 6 <= last_order_date` (#28).
- 4.16 is_provisional = `week_start + 6 + return_window_days > last_order_date` (#24).
- 4.17 is_as_of_week is true on exactly one row: the latest complete week (#11, #33).

Comparison and placement
- 4.18 prior_<m> comes from a join on `prior.week_start = cur.week_start - 7` (DATE minus integer), never row order. change_<m> = <m> - prior_<m>, using stored (rounded) values; null if either side is null. Both null in the first week (#31).
- 4.19 All metric logic lives in models. The mart is final; the dashboard selects from it and computes nothing (#17).

Sources and joins
- 4.20 Raw CSVs are declared as dbt-duckdb external sources in the staging sources YAML; models read them only through `source()`. No seeds; no file paths in model SQL (#1, #37).
- 4.21 Staging casts every kept column explicitly to the types in S-1 to S-3. Timestamps are plain TIMESTAMP, never TIMESTAMPTZ (#37).
- 4.22 int_order_items joins stg_orders (inner; F-2 guarantees a match) and stg_products (left), so a missing product never drops revenue (#37).

## 5. Configuration

| Item | Where | Value | Rule |
|---|---|---|---|
| return_window_days | dbt var, dbt_project.yml | 11 | 4.16, T-11 (#24) |
| Materialization | dbt_project.yml | staging: view; intermediate, marts: table | #14 |
| Source locations | sources YAML in models/staging/ | the four CSVs in data/raw/ | 4.20 |

Status literals ('Cancelled', 'Returned') stay in SQL, guarded by T-5. Expected test values are snapshot facts, not config; they change only through a logged decision (8.4).

## 6. Failure behavior

- 6.1 No loader, so no quarantine. Bad input surfaces as a failed `dbt build`.
- 6.2 Severity: T-3 and T-11 warn (#13, #24). Every other test errors. An error blocks downstream models, so the mart cannot build on a failed upstream check.
- 6.3 Reruns are full rebuilds and idempotent: views recreated, tables replaced. Two consecutive builds produce identical marts.
- 6.4 If T-6, T-10, or T-14 fails by cents, stop and report the difference. Do not change the cast or add a tolerance (#32).
- 6.5 If any rule or pseudocode cannot be implemented as written, stop and report. Do not improvise.

## 7. Acceptance tests

Expected values are from discovery against the current snapshot. Singular tests live in tests/; schema tests and unit tests in YAML next to their models (structure.md).

Source and staging

| # | Test | Type | Expected |
|---|---|---|---|
| T-1 | unique and not_null: stg_orders.order_id, stg_order_items.order_item_id, stg_products.product_id, source users.id | schema | pass (F-1) |
| T-2 | relationships: stg_order_items.order_id -> stg_orders.order_id; stg_order_items.product_id -> stg_products.product_id; stg_orders.user_id -> source users.id | schema | pass (F-2) |
| T-3 | count of source order_items whose user_id differs from their source order's user_id; reads source() only | singular | = 4 (F-4); warn if not |
| T-4 | count of source order_items whose status differs from their source order's status; reads source() only | singular | = 0 (F-6) |
| T-5 | accepted_values on stg_orders.status | schema | Processing, Shipped, Complete, Cancelled, Returned (F-5) |
| T-6 | stg_order_items row count and SUM(sale_price) | singular | 180,952 and 10,758,880.13 (F-10) |
| T-7 | stg_orders row count | singular | 124,923 |

Intermediate

| # | Test | Type | Expected |
|---|---|---|---|
| T-8 | int_order_items: order_item_id unique and not_null; row count | schema + singular | 180,952 |
| T-9 | int_orders: order_id unique and not_null; row count; item_count = num_of_item on every row | schema + singular | 124,923 rows; 0 mismatches (F-3) |
| T-10 | SUM(int_orders.order_sale_price) | singular | 10,758,880.13 |
| T-11 | for orders with returned_at: `date_diff('day', order_created_at, returned_at) <= return_window_days` (method of Q16) | singular | 0 violations (F-20); warn |
| T-12 | for qualifying orders, first_qualifying_order_at <= order_created_at; first_qualifying_order_at is null only for users with no qualifying order | singular | 0 violations |

Mart

| # | Test | Type | Expected |
|---|---|---|---|
| T-13 | week_start unique and not_null; row count; min and max week_start | schema + singular | 179 rows; 2019-01-07 to 2022-06-06 (F-22) |
| T-14 | SUM(gross_revenue) over all weeks | singular | 10,758,880.13 (F-10) |
| T-15 | net revenue reconciliation (#21): from source() only, sum sale_price (each cast per 4.5) for orders not Cancelled or Returned, grouped by `date_trunc('week', orders.created_at)::DATE`; full outer join to the mart; mart weeks without raw rows must have net_revenue = 0 | singular | 0 weeks differing; exact match (F-22) |
| T-16 | week 2022-05-30: customers, new_customers, returning_customers | singular | 3,601 / 2,361 / 1,240 (F-21, #27) |
| T-17 | new_customers + returning_customers = customers on every row | singular | 0 violations (#25) |
| T-18 | SUM(orders) over all weeks | singular | 106,200 (F-22) |
| T-19 | SUM(new_customers) over all weeks | singular | 71,951 (F-22); each customer is new exactly once |
| T-20 | flags per 7.1; last_order_date = 2022-06-08 on every row; is_as_of_week true on exactly one row, 2022-05-30 | singular | 0 violations |
| T-21 | for every week after the first, each prior_<m> equals <m> of the row whose week_start is 7 days earlier; in the first week all prior_ and change_ columns are null | singular | 0 violations |

7.1 Expected flags (T-20)

| week_start | is_complete | is_provisional | Why |
|---|---|---|---|
| 2019-01-07 | true | false | first week counts as complete (#28) |
| 2022-05-16 | true | false | 05-22 + 11 = 06-02 <= 06-08 |
| 2022-05-23 | true | true | 05-29 + 11 = 06-09 > 06-08 |
| 2022-05-30 | true | true | as-of week; always provisional (#33) |
| 2022-06-06 | false | true | partial week (F-15) |

Unit tests (dbt unit tests, fixtures use only columns in section 3)

| # | Model | Case | Expected |
|---|---|---|---|
| U-1 | int_orders | user's first order Cancelled, second Complete | first_qualifying_order_at = second order's created_at |
| U-2 | int_orders | user with only Cancelled orders | first_qualifying_order_at null; is_qualifying false |
| U-3 | int_order_items | orders at Sunday 23:59:59 and Monday 00:00:00 UTC | different week_start |
| U-4 | mart_weekly_metrics | a week with zero qualifying orders | orders 0; aov null; next week's prior_orders = 0, not null |
| U-5 | mart_weekly_metrics | customer whose first qualifying order was in an earlier week | counted in returning_customers, not new_customers |

## 8. Operational notes

- 8.1 Build: from the repo root, `uv run dbt build`. Builds all models and runs all tests in DAG order against dev.duckdb on full data (#18).
- 8.2 Reprocess: rebuild. Nothing is incremental. A return-policy change is an edit to `return_window_days` and a rebuild.
- 8.3 Monitor: warnings on T-3 and T-11 mean the data changed shape. Log and decide.
- 8.4 Data refresh: replace the CSVs in data/raw/ and rebuild. These snapshot expectations fail by design and change only through a logged decision: T-3, T-6, T-7, T-8, T-9, T-10, T-13, T-14, T-16, T-18, T-19, T-20.
- 8.5 Test file naming: `tests/assert_<subject>.sql`, with the T-number and expected-value citation (F- or #) in a header comment. Expected values are literals in the test.

## 9. Decisions log

Decisions live in docs/decisions.md. This spec implements #10-#13, #21, #24-#28 and #29-#37.

| Version | Date | Change |
|---|---|---|
| v1.0 | 2026-10-27 | First version. |
| v1.0.1 | 2026-10-27 | Removed references to chat-only working numbers (AM-, Q-); environment open items stated in full as E-1 and E-2. No rule or test changed. |
| v1.0.2 | 2026-10-27 | E-1 and E-2 answered: no project files, dbt not installed; SETUP-01 creates them. No rule or test changed. |
