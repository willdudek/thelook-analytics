# thelook-analytics -- Strategy brief

Stage 3 (Strategy) of the Two Brains loop. Completed 2026-10-16.
Evidence references (F-, C-, K-) point to docs/discovery.md; decisions (#) to docs/decisions.md.

Problem: The founder of a small DTC apparel brand runs a Monday business review to decide where to push and where to pull back: marketing spend, promotions, and reorders. Today those decisions rest on numbers that don't agree. Revenue, orders, and customer counts differ depending on which export someone pulled and how they filtered it. When a week is up or down, nobody can say whether it was more customers, bigger baskets, or fewer returns. We will build one weekly view covering sales, customers, and cancellations/returns, with every metric defined in writing and tested against the source data. When this works, the founder gets the same number every time it's pulled and can see what drove the change, so the meeting moves from arguing about numbers to acting on them.

Success looks like:
1. Same number every pull. A clean clone plus `dbt build` reproduces the marts exactly, and the marts reconcile to raw by independent paths: gross revenue summed over all weeks equals the raw total of 10,758,880.13 (F-10), and mart net revenue per week equals a raw-side sum of sale_price for orders not Cancelled or Returned, grouped by order week (#10, #21).
2. The change is explained. For the as-of week (2022-05-30, #11) the founder's page shows the revenue waterfall alongside orders, customers, and average order value, so "more customers, bigger baskets, or fewer returns" is answerable from one page.
3. One hand-verified number for one week, computed by an independent path, matches the mart exactly.
4. The loop is practiced (#7): every dispatch has a definition of done and a review against the checklist, and every decision is logged.

Constraints:
1. Synthetic data (C-1). Patterns, including the June 3-6 spike (F-15), are not evidence of customer behavior; nothing is concluded from them.
2. Static snapshot (C-2). "This week" is a rule applied to fixed data (#11).
3. Revenue is sale_price only; no tax, shipping, or discount columns (F-12, C-3). Money is stored as floating point and must be cast (F-11).
4. Timestamps carry no timezone (F-18); assumed UTC (#12).
5. Stack: Mac, zsh, uv, dbt-duckdb with local dev.duckdb, Evidence, GitHub (#2, #17). Free tools only.
6. Solo, learning-only project (#7); no deadline.

Options considered:
  A. Core then retention, two specs on one layered design (L-D, #14).
     Spec 1: staging, item and order intermediates, weekly mart, Evidence page.
     Spec 2: customer-month intermediate, cohort retention and churn marts, a second page.
     Pros: each spec is small enough to run the full loop; Spec 2 builds on reviewed, reconciled layers; rules are applied in one model.
     Cons: two passes through stages 4-7; retention arrives later.
     Effort: roughly 5-6 dispatches for Spec 1 (setup, staging, intermediate, mart, reconciliation, dashboard), 3 for Spec 2.
  B. Everything in one spec.
     Pros: one spec, one review cycle.
     Cons: two grains (week and customer-month) in a first spec; a staging bug would only surface after retention is built on top of it; the spec is too large to review well.
     Effort: similar dispatch count, higher rework risk.
  C. Minimal: order grain only, weekly metrics, no retention (L-A).
     Pros: fastest to a working dashboard.
     Cons: drops churn and retention, which the founder wants (#16); category would later need an item-grain path anyway.
     Effort: roughly 4 dispatches.

Recommendation: A, because it delivers every metric the founder asked for while keeping each spec small enough to verify, and it applies each data rule (#10, #12, #13) in exactly one model, which directly addresses the "numbers don't agree" problem.

Key choices made in this stage:
  - Revenue: net booked revenue by order week, shown as a waterfall (#10)
  - As-of week: latest complete ISO week, currently 2022-05-30 (#11)
  - Timezone: UTC (#12)
  - Customer of record: orders.user_id (#13)
  - Layers: L-D, built as Spec 1 and Spec 2 (#14)
  - Scope: category to v1.1 (#15); churn and retention in v1 via Spec 2 (#16)
  - Dashboard: Evidence, marts-only queries (#17)
  - Structure: dbt at repo root, Evidence in dashboard/ (#18-20)

What would change this:
1. Revenue must tie to the books or to cash, or a published week must never change: revenue moves to event-dated (K1-C).
2. The brand becomes real with a home timezone, or a stakeholder asks for local weeks: convert in staging (K4-B).
3. Many user mismatches appear instead of one (F-4): attribution needs a stakeholder decision.
4. A Node toolchain proves unworkable: dashboard moves to Streamlit (D-C).
5. Spec 1 review shows the intermediate design is wrong: fix it before Spec 2 starts.

Risks and mitigations:
1. Definition drift. "Active" now means an order in the last 6 months (#16), while the weekly view also needs a this-week customer count. Mitigation: Spec 1's metric table gives each its own name and one definition; dashboards select from marts only (#17).
2. Reading meaning into synthetic data (C-1, F-7, F-15). Mitigation: caveat in the README and on the dashboard; the spike is a test case for decomposition, not a finding.
3. Scope creep. Churn and retention already entered v1 (#16); category and funnel are queued (#15, #6). Mitigation: two specs; an explicit out-of-scope list and stop condition in every dispatch; v1.1 items stay in the log, not in dispatches.

Open questions:
1. Return lag and the provisional window (K-3). Query in stage 4. Will.
2. Customer definitions for Spec 1 (K-6): the name for this-week customers vs 6-month active; new vs returning; whether cancelled orders count as activity. Will.
3. Event time for item-level metrics (K-7); proposed orders.created_at (F-16). Spec 1. Will.
4. Hand-verified number: which week, which metric, and the independent computation path. Spec 1. Will.
5. How the partial week of 2022-06-06 appears in the marts (F-15). Spec 1. Will.
6. Spec 2 definitions: LTV basis (proposed: net revenue per #10), handling of cohorts too young to have M1-M3, churn as-of logic with a 6-month window. Will.
7. Does the Kaggle page document the source timezone (#12)? Optional check. Will.
8. Which Evidence component renders the cohort grid. Check Evidence docs before Spec 2. Agent or Will.