# Decisions log

| # | Date | Decision | Why | Who |
|---|------|----------|-----|-----|
| 1 | pre-2026-10-15 | Data source: TheLook eCommerce Kaggle snapshot; raw CSVs in data/raw/, gitignored | Realistic DTC-shaped tables; free; no data in git | Will |
| 2 | pre-2026-10-15 | Storage: local DuckDB (dev.duckdb) via dbt-duckdb; MotherDuck maybe later as prod | Zero-cost local dev; matches existing skills | Will |
| 3 | pre-2026-10-15 | Dashboard tool undecided; leaning Evidence | Dashboard as code in repo, reviewable as a diff | Will |
| 4 | pre-2026-10-15 | v1 kept small: core metrics, staging→intermediate→marts, schema tests, one reconciliation test, one hand-verified number, one dashboard page; messy subscriptions file is v2 | Practice the full loop before adding scope | Will |
| 5 | 2026-10-15 | Primary audience for v1 is the founder; marketing and ops views later | One decision-maker keeps the metric list short | Will |
| 6 | 2026-10-15 | v1 scope: sales, customers, cancellations/returns (~6 metrics). Funnel = v1.1; margin pending discovery; inventory, MotherDuck, scheduling, loader out | Uses one grain (orders/items/customers); funnel adds a new grain | Will |
| 7 | 2026-10-15 | Purpose: learning only; synthetic-data caveat noted in README as context | No portfolio polish needed | Will |
| 8 | 2026-10-15 | No house metric definitions carried in; derive in discovery and spec | Avoid assuming columns before profiling | Will |
| 9 | 2026-10-15 | Stage-1 framing paragraph locked (see docs/strategy-brief.md) | Becomes the Problem field of the strategy brief | Will |