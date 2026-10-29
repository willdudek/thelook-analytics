# Agent instructions: thelook-analytics

## Role
- You implement dispatches. Design decisions are made outside this session and live in docs/specs/ and docs/decisions.md. Do not decide what to build.
- The repo files are the source of truth. If a dispatch prompt conflicts with a spec, structure.md, or this file, follow the file and report the conflict.

## Files
- Read docs/structure.md before creating any file. Do not create top-level files or folders not listed there without asking. Tool-generated folders listed there are allowed.
- Never read, quote, or act on notes.txt. It is the owner's scratch file (#20).
- data/raw/ is read-only. Never modify, copy, or move raw files.
- Do not run exploratory queries over full raw tables. Work from the models and files named in the dispatch, and use dbt unit tests for logic checks.

## Building
- Run everything from the repo root through uv: `uv run dbt ...`.
- Never invent column names, table names, or definitions. If a rule or pseudocode cannot be implemented as written, stop and report. Do not improvise (spec 6.5).
- No new dependencies or dbt packages without asking.
- Do not edit anything outside the repo (~/.dbt/, global installs, shell config).

## Stop conditions
- If any step fails twice with the same error, stop and report the full error output. No workarounds.
- If dbt reports dev.duckdb is locked, stop and report. Do not delete, move, or rename dev.duckdb or change the profile path.
- If a test fails by cents on a money total, stop and report the difference. Do not change casts or add tolerances (spec 6.4).

## Expected warnings (report, do not fix)
- "Configuration paths exist in your dbt_project.yml file which do not apply to any resources" while a layer has no models yet.

## Finishing
- Show evidence: the command output for every definition-of-done item.
- Do not commit or push. The owner reviews first.
- Report: files created or changed; ambiguities in the spec; surprises in the data or tools; what you would do next (do not do it).