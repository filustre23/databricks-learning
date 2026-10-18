---
name: dbt-dev-loop
description: Scoped dbt development loop on Databricks — build and test selected models into lab_dev (via the dbt MCP or uv run dbt), check contracts and unit tests, compare row counts and key metrics against lab_prod, and review the user's SQL.
argument-hint: "<selector, e.g. stg_source_b__eligibility+ | tag:source_c+>"
---

# /dbt-dev-loop

The project is `bundles/transform/dbt`, target `dev`, which writes to `lab_dev.dbt_<user>_*` and reads prod
`<source_id>_bronze` schemas. Staging and intermediate models build into `silver`; marts build into `gold`.

1. **Scope.** Require a selector; refuse a bare `dbt build`. For anything wider than one source's graph, run
   `/cost-check` first.
2. **Review first.** Read the models the user changed in the selection. Check:
   - **Grain**, with the PK test matching it.
   - **Typing** of dates and amounts.
   - **The adjustment/void logic.**
   - **Incremental strategy:** `merge` with a `unique_key` vs `replace_where`, and an `is_incremental()`
     predicate on `_ingested_at`.
   - **Contracts on marts.**
   - **Hard-coded catalogs.** There should be none; use `source()` and `ref()`.

   Give feedback before running anything.
3. **Compile and parse.** Run them through the dbt MCP (or `uv run dbt compile -s <sel> --target dev`). Fix errors first.
4. **Build.** dbt MCP `build`, or `uv run dbt build -s <sel> --target dev`. This writes to dev only. Never use
   `--target preprod` or `--target prod` locally: those are built only by CI on `main` and `release`.
5. **Read results.** Summarize failures by type (unit test, contract, data test). For data tests with
   `store_failures`, query `lab_dev.dbt_test_failures.<test>` for a few example rows and explain the root cause.
6. **Compare to prod** (when prod exists): row counts and key aggregates per model
   (`count(*)`, `count(distinct member_id)`, `sum(paid_amount)`, `min/max(service_date)`) in `lab_dev` vs `lab_prod`. After a merge to `main`, run
   the same comparison for `lab_preprod` vs `lab_prod`; that's the evidence for the `main` → `release` PR.
   Explain every difference: intended or a bug.
7. **Freshness.** For staging changes, also run `dbt source freshness -s source:<source_id>`.
8. **Done.** Everything is green, the diff is explained, and the user commits and opens a PR to `main`.
   Promotion continues with `/bundle-ship transform preprod`, then `prod`. Log notable findings in `docs/progress.md`,
   and dbt-databricks quirks in `docs/evaluation.md`.
