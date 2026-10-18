---
name: add-payer-files
description: Runbook for landing payer files. Mode A lands a new drop for an existing source; mode B onboards a brand-new source end to end (registry YAML, landing volume, bronze, dbt staging, governance, alerts). Walks a checklist one step at a time and verifies each box before moving on.
argument-hint: "<source_id> [feed] [yyyy-mm-dd] | new <source_id>"
---

# /add-payer-files

Walk the user through landing payer files with a checklist. **One step at a time:** state the step,
give the command (the user runs mutating ones in their own tab), wait for the pasted output, verify it,
tick the box, then move on. Never skip a box. If a check fails, stop and switch to `/triage-run`.

## Pick the mode from $ARGUMENTS

- `new <source_id>`, or a `source_id` with no `config/sources/<source_id>.yml` → **Mode B (onboard)**
- An existing `source_id` → **Mode A (new drop)**
- Nothing given → list `config/sources/*.yml` and ask which source and which mode

Start by printing the full checklist for the chosen mode with empty boxes, so the user sees the whole path.

## Mode A — new drop for an existing source

1. [ ] **Budget.** Run `/cost-check`. If over $325, stop.
2. [ ] **Identify.** Confirm `source_id`, feed(s) and drop date. Read `config/sources/<source_id>.yml` and
   show the expected format, delimiter, cadence, `expected_columns`, PK and expectations.
3. [ ] **Naming.** Check every local file name matches
   `<source_id>_<feed>_<yyyymmdd>[_<seq>].<ext>` and the target prefix
   `s3://<landing-bucket>/<source_id>/<feed>/yyyy/mm/dd/`. Flag any mismatch.
4. [ ] **Local sanity** (Claude may run these, read-only):
   - Row count, header and delimiter or encoding (`head`, `wc -l`, `gzip -t` for `.gz`, `uv run python -c` for Parquet row counts).
   - Header columns against the registry's `expected_columns`; report what's missing or extra.
   - A hash of each file, checked against `observation.file_ledger` for duplicate re-drops.

   Record the expected row count per file, because step 8 checks against it.
5. [ ] **Drop.** Hand the user the `aws s3 cp` command (with `--profile` and `--region us-west-2`). Then verify:
   - `aws s3 ls s3://<landing-bucket>/<source_id>/<feed>/yyyy/mm/dd/`
   - `databricks fs ls dbfs:/Volumes/lab_prod/landing/<source_id>/<feed>/yyyy/mm/dd/ -p lab`
6. [ ] **Trigger.** Check the file-arrival trigger fired: `databricks jobs list-runs --job-id <ingest_job> -p lab`
   should show a run that started after the drop. If none appears within a few minutes, hand over `databricks jobs run-now`.
7. [ ] **Pipeline update.** Check the update reached COMPLETED: `databricks pipelines list-updates <pipeline_id> -p lab`.
   If it FAILED, stop and run `/triage-run`.
8. [ ] **Row parity.** Bronze rows for these files plus quarantine rows must equal the source rows from step 4.
   Query `observation.row_parity`, or count directly by `_source_file`.
9. [ ] **Quality.**
   - `_rescued_data` is null for these files.
   - No `expect_or_fail` trips in `observation.expectations`.
   - Quarantine stays under the registry threshold; show counts per rule.
   - The volume is within band in `observation.feed_volume`.
10. [ ] **Post-ingest check task** in the job run is green.
11. [ ] **Downstream dbt.** Run `/dbt-dev-loop tag:<source_id>+`, or confirm the job's dbt task passed for prod.
12. [ ] **Freshness.** `observation.feed_freshness` shows this source and feed as current.
13. [ ] **Log** to `docs/progress.md`: the date, source, feed, files, rows, quarantined rows, any anomalies, and pass/fail.

## Mode B — onboard a new source

1. [ ] **Budget.** Run `/cost-check`.
2. [ ] **Identify.** Pick a `source_id` and the list of feeds, and agree the format and cadence per feed.
3. [ ] **Get sample files** and check their naming as in Mode A step 3.
4. [ ] **Profile.** Give the user `read_files(...)` SQL to run in the SQL Editor against a scratch upload,
   or profile locally. Record the quirks: header and delimiter, date formats, wide or long diagnosis columns,
   claim header/line grain, adjustment and void indicators, member key, PHI columns.
5. [ ] **Registry.** The user writes `config/sources/<source_id>.yml`: feeds, format, delimiter, `path_glob`,
   cadence, `freshness_sla`, `expected_columns`, PK, `phi_columns`, `schema_evolution_mode`, the three
   expectation tiers, and the quarantine threshold. Claude reviews it against the profile from step 4.
6. [ ] **Unit test.** Add a registry test or a parser test if the format is new (fixed-width, FHIR, X12).
   Run `uv run pytest tests/unit -q`.
7. [ ] **Golden drop.** Build `tests/fixtures/<source_id>/` with known good rows and known violations of each
   rule, plus the expected counts.
8. [ ] **Landing infra.** The user adds a `source_landing` module call (S3 prefix, external volume, the `<source_id>_bronze`
   schema, grants) in
   `terraform/30-unity-catalog`. Run `/tf-stack 30`.
9. [ ] **Bronze.** Add the source's pipeline to the bundle. It uses the same generic `bronze_from_registry.py`, parameterized by
   `source_id`, and publishes to `<catalog>.<source_id>_bronze`. The registry YAML plus one resource entry should be the only changes.
   Run `/bundle-ship ingestion dev` against the golden drop in `lab_dev.landing`, then run the golden-drop integration
   test and require exact counts.
10. [ ] **Land the real sample** (Mode A steps 5–10).
11. [ ] **dbt staging (silver).** The user writes the `sources.yml` entry pointing at `<source_id>_bronze` (with `loaded_at_field: _ingested_at` and freshness
    `warn_after`/`error_after` taken from the cadence), the `stg_<source_id>__<feed>` models, and tests.
    Claude reviews the column mapping against the common contract (`docs/data-contracts.md`).
12. [ ] **Union (silver).** Add the source to `union_sources()` so it joins the conformed `int_*` models, then run `/dbt-dev-loop tag:<source_id>+`. Contracts must pass.
13. [ ] **Governance.** Extend the row filter to the new `source_id`, add a `source_ops_<x>` group if needed,
    and tag `phi=true` on the PHI columns. Verify a restricted principal sees only filtered or masked rows.
14. [ ] **Alerts.** Add the source and feeds to the freshness SLA and volume bands. Confirm the post-ingest check
    task covers them. Run at least the `missing_column` and `bad_rows` drills (`/break-it`) against this source.
15. [ ] **Docs.** Update `docs/onboarding-a-new-source.md` with anything this source taught you.
16. [ ] **Ship.** Open a PR to `main`. Once merged, run `/bundle-ship ingestion preprod` and `/bundle-ship transform preprod`.
    Preprod must be green, with the golden drop passing in `lab_preprod`. Then promote with the `main` → `release` PR:
    `/bundle-ship ingestion prod`, then `/bundle-ship transform prod`. Real files land only after the prod deploy.
17. [ ] **Close out.** Run `/phase-check` and log the result to `docs/progress.md`.

## Rules

- Synthetic data only, region us-west-2, serverless only.
- Mutating commands go to the user. Claude runs read-only commands and SELECTs.
- Every count must reconcile: source rows = bronze rows + quarantine rows. Explain any gap before ticking the box.
