# Repo structure

Project structure P-A (#18). One line per item: what it is responsible for.
Model names are set in the specs (#14), not here.

```
thelook-analytics/
├── CLAUDE.md            Agent instructions: roles, rules, files not to read (#20)
├── README.md            Purpose, setup steps, synthetic-data caveat (#7)
├── .gitignore           Excludes data/, dev.duckdb, notes.txt, dbt target/ and logs/ (#1, #19, #20)
├── pyproject.toml       Python dependencies (dbt-duckdb), managed by uv
├── uv.lock              Exact pinned versions so a clean clone builds the same (success #1)
├── dbt_project.yml      dbt config: model paths, materialization per layer (#14)
├── profiles.yml         Connection to local dev.duckdb; no secrets (#18)
├── notes.txt            Owner scratch; gitignored; agent must not read (#20)
├── data/                Gitignored (#1)
│   ├── raw/             Kaggle CSVs, unmodified
│   └── samples/         Sample rows for the chat (#19)
├── docs/
│   ├── discovery.md     Stage 2 findings (F-, C-, K-)
│   ├── strategy-brief.md  Stage 3 brief
│   ├── decisions.md     Decisions log; one row per decision
│   ├── structure.md     This file
│   └── specs/           One spec per component (Spec 1, Spec 2)
├── models/
│   ├── staging/         One view per raw table: casts, renames, drops PII; source definitions (#14, F-11, F-19)
│   ├── intermediate/    Item-grain and order-grain tables; attribution rules applied once (#13, #14)
│   └── marts/           Weekly metrics (Spec 1); retention and churn (Spec 2) (#14, #16)
├── tests/               Singular SQL tests: reconciliation and hand-verified number (#21)
└── dashboard/           Evidence project; queries select from marts only (#17)
```

Unit tests (#18) are defined in YAML next to the model they test, not in tests/.