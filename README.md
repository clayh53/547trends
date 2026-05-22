# 547trends

Daily production dashboard for HI-A547-C, sourced from Arena Energy DPRs.

- `index.html` — dashboard (renders at https://clayh53.github.io/547trends/)
- `arena_production.csv` / `.json` — extracted daily allocated volumes
- Data refreshes via a local Mac launchd job that pulls new PDFs from Gmail,
  extracts the values, and pushes here.
