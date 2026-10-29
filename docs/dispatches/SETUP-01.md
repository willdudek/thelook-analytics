SETUP-01: Python + dbt project scaffold

Read first: CLAUDE.md, docs/structure.md, docs/specs/weekly-metrics.md
sections 2.4, 3.1, and 5.

Build:
- A uv project at the repo root, created with `uv init --bare`. Pin Python
  with `uv python pin 3.12` (writes .python-version); requires-python
  >= 3.12. Add exactly two dependencies: dbt-core >= 1.8 and dbt-duckdb.
  uv may download a managed Python 3.12; that is allowed.
- dbt_project.yml at the repo root. Project name thelook_analytics, profile
  thelook_analytics. Materializations: staging = view, intermediate = table,
  marts = table (spec section 5). Var return_window_days: 11.
- profiles.yml at the repo root. Profile thelook_analytics, target dev,
  type duckdb, path dev.duckdb, settings TimeZone: UTC (spec 2.4, #40).
  No secrets.
- Replace .gitignore with exactly these lines, in this order:
    data/
    *.duckdb
    *.duckdb.wal
    .env
    .venv/
    .DS_Store
    notes.txt
    target/
    logs/
    dbt_packages/
    .user.yml

Do not:
- Run dbt init, or create any example models, README.md, main.py, or src/.
- Create sources YAML, models, tests, seeds, packages.yml, or anything in
  dashboard/. Those belong to later dispatches.
- Add any dependency beyond dbt-core and dbt-duckdb.
- Touch data/ or ~/.dbt/.

Definition of done (show the output of each):
1. uv run dbt --version shows dbt-core >= 1.8 and the duckdb plugin.
2. uv run dbt debug ends with "All checks passed!", and its output shows
   profiles.yml loaded from the repo root, not ~/.dbt/.
3. uv run dbt show --inline "select current_setting('TimeZone') as tz"
   returns UTC. The "Configuration paths ... do not apply" warning is
   expected here.
4. git status --short shows exactly: modified .gitignore; new
   .python-version, pyproject.toml, uv.lock, dbt_project.yml, profiles.yml.
   Nothing else.
5. git check-ignore -v dev.duckdb dev.duckdb.wal .venv target logs .user.yml
   lists all six as ignored.

Stop condition: if any step fails twice with the same error, stop and
report the error output. Do not try workarounds such as a different
Python, global installs, or editing ~/.dbt/.

Report back: files created or changed; versions installed; any conflict
with structure.md or CLAUDE.md; anything surprising; what you would do
next (but do not do it). Do not commit; I will review first.


---

## Result (reviewed 2026-10-29)
Verdict: accepted with follow-ups.
DoD 1-5: pass. dbt-core 1.12.6, dbt-duckdb 1.11.1, DuckDB 2.0.0; profiles.yml loaded from repo root; TimeZone UTC; git status and check-ignore as specified.
Agent deviations (accepted): removed uv's >=1.11.1 bound on dbt-duckdb; added version and config-version to dbt_project.yml.
Process gap: agent's log showed 1 file read of the 3 required. Future dispatches add a DoD line listing files read.
Follow-ups: DuckDB CLI mismatch (resolved, #47); README.md (#46).
Commit: 70a6d50