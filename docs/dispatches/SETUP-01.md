SETUP-01: Python + dbt project scaffold

Read first: CLAUDE.md, docs/structure.md, docs/specs/weekly-metrics.md
sections 2.4, 3.1, and 5. If structure.md or CLAUDE.md conflicts with this
prompt on a file location or name, follow the file and report the conflict.

Build:
- A uv project at the repo root: pyproject.toml, uv.lock, .python-version
  pinned to 3.12. Dependencies: dbt-core >= 1.8 and dbt-duckdb. Nothing else.
- dbt_project.yml at the repo root. Project name thelook_analytics, profile
  thelook_analytics. Materializations: staging = view, intermediate = table,
  marts = table (spec section 5, #14). Var return_window_days: 11.
- profiles.yml at the repo root. Profile thelook_analytics, target dev,
  type duckdb, path dev.duckdb (relative to the repo root). No secrets.
- Make sure .gitignore covers dev.duckdb, .venv/, target/, logs/, and
  dbt_packages/. Add only the missing lines.

Do not:
- Run dbt init, or create any example models.
- Create sources YAML, models, tests, seeds, packages.yml, or anything in
  dashboard/. Those belong to later dispatches.
- Add any dependency beyond dbt-core and dbt-duckdb.
- Touch data/raw/.

Definition of done (show the output of each):
1. uv run dbt --version shows dbt-core >= 1.8 and the duckdb plugin.
2. uv run dbt debug ends with "All checks passed!"
3. git status shows only the new and changed files listed above. dev.duckdb
   does not appear.
4. git check-ignore dev.duckdb prints dev.duckdb.

Stop condition: if any step fails twice with the same error, stop and
report the error output. Do not try workarounds such as a different
Python, global installs, or editing ~/.dbt/.

Report back: files created or changed; versions installed; any conflict
with structure.md or CLAUDE.md; anything surprising; what you would do
next (but do not do it). Do not commit; I will review first.