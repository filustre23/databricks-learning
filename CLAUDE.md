# CLAUDE.md

Standalone Databricks-on-AWS learning lab. Several synthetic payer sources drop differently shaped
eligibility and claims feeds; they land in S3, surface as Unity Catalog volumes, stream into a
per-source bronze schema with Lakeflow pipelines (Auto Loader), come together in silver, where dbt-databricks
conforms and unions them, and are served from gold as one common claims and eligibility model, orchestrated by Lakeflow Jobs and shipped with Asset Bundles + Terraform + GitHub Actions.

## Working agreement

**The user writes all lab code and runs every command that changes state. Claude coaches, reviews and verifies.**

- Claude may run read-only commands itself: `databricks … list|get`, `databricks bundle validate|summary`,
  `databricks fs ls`, `terraform fmt -check|validate|plan`, `aws s3 ls`, `aws sts get-caller-identity`,
  and SELECT queries.
- Claude **prints** mutating commands for the user to run in their own terminal tab, then verifies the
  result: `terraform apply|destroy|import`, `databricks bundle deploy|run|destroy`, `databricks fs cp|rm`,
  `databricks jobs run-now|repair-run`, `databricks pipelines start-update`, `aws s3 cp|mv|rm|sync`, `git push`.
  `.claude/settings.json` denies these.
- Don't scaffold lab code (Terraform, pipelines, dbt models, generators). Review what the user writes,
  explain why, and point to the docs. Claude tooling (`CLAUDE.md`, `.claude/`, `.mcp.json`) and
  `docs/progress.md` entries are the exception.
- Work one step at a time: give the next command, wait for the pasted output, then verify.

## Hard rules

- **Standalone.** Every resource comes from this repo's `terraform/` stacks. No `data` sources or imports
  pointing at pre-existing VPCs, buckets, roles or state, and no connections to other warehouses.
- **Region `us-west-2` only.** Pin it in every AWS provider block (the local AWS config defaults to us-east-1).
- **Serverless only.** Use a serverless workspace (`compute_mode = "SERVERLESS"`), serverless SQL warehouse,
  serverless jobs and triggered serverless pipelines. No classic or all-purpose clusters, no continuous
  pipelines, no model serving endpoints.
- **Budget cap $400**, which is the trial's credits. Tripwires are at **$150 / $250 / $325**; at $325, stop
  and run `/teardown`. Run `/cost-check` before anything expensive: a full `dbt build`, a backfill, a
  pipeline full refresh, or a new dashboard or Genie session.
- **Synthetic data only.** Never load real claims or PHI.
- **Public repo.** Never commit account IDs, ARNs, workspace URLs, tokens, `*.tfvars`, Terraform state,
  `.databrickscfg` or a real `profiles.yml`. Commit `.example` files instead.
- **Tags.** Every AWS resource gets `project=databricks-learning`, `owner`, `ttl` via provider `default_tags`.

## Layout

| Path | Holds |
|---|---|
| `terraform/00-bootstrap` | State bucket, AWS budget, GitHub OIDC → AWS role |
| `terraform/10-workspace` | Serverless workspace, Databricks account budget |
| `terraform/20-account` | Metastore assignment, groups, service principals |
| `terraform/30-unity-catalog` | Landing/outbound buckets, storage credential, external locations, catalogs, schemas, volumes, grants |
| `terraform/40-workspace-config` | SQL warehouse, serverless budget policy, secret scope |
| `terraform/modules/` | `source_landing`, `uc_env_catalog`, `service_principal` |
| `generators/` | Local scripts that produce each source's files; `drop_simulator.py` replays them on a cadence |
| `config/sources/*.yml` | The feed registry: one file per source (formats, paths, cadence, keys, PHI columns) |
| `libs/source_ingest/` | Shared Python wheel (registry loader, fixed-width / FHIR / X12 parsers) |
| `bundles/ingestion/` | Bundle: one Lakeflow bronze pipeline per source (shared generic code, driven by the registry) + file-arrival ingest jobs |
| `bundles/transform/` | Bundle: dbt project (`dbt/`) + dbt job |
| `bundles/ops/` | Bundle: dashboards, alerts, Genie space |
| `tests/` | `unit/` (pytest), `integration/` (golden-drop pipeline test), `fixtures/<source_id>/`, `drills/` |
| `docs/` | Architecture, ADRs, onboarding runbook, data contracts, `progress.md`, `evaluation.md`, `gcp-to-databricks.md` (GCP → Databricks equivalents) |

## Naming

- Sources: `source_a` (CMS Medicare FFS RIF, pipe-delimited, monthly), `source_b` (Tuva Aetna layout,
  CSV.gz + fixed-width eligibility, weekly), `source_c` (Tuva HCCI layout, Parquet, daily). Stretch:
  `source_d` (Blue Button FHIR NDJSON), `source_e` (X12 or Synthea).
- Feeds: `eligibility`, `medical`, `medical_ip`, `medical_op`, `professional`, `pharmacy`.
- Landing: `s3://<landing-bucket>/<source_id>/<feed>/yyyy/mm/dd/<source_id>_<feed>_<yyyymmdd>[_<seq>].<ext>`,
  surfaced as `/Volumes/lab_prod/landing/<source_id>/<feed>/…`.
- Catalogs: `lab_dev`, `lab_preprod`, `lab_prod`. dbt dev output: `lab_dev.dbt_<user>_*`. CI output: `lab_dev.ci_pr<N>_*`.
- Schemas (medallion: each source stays separate in bronze; the sources come together in silver):

  | Schema | Holds | Owner |
  |---|---|---|
  | `landing` | One external volume per source (raw files, immutable) | Terraform |
  | `<source_id>_bronze` (`source_a_bronze`, `source_b_bronze`, …) | That source's raw streaming tables, one per feed: `<feed>`, plus `<feed>_quarantine` | That source's Lakeflow pipeline |
  | `silver` | dbt `stg_<source_id>__<feed>` (per-source rename and cast, as views), then `int_*`. **The sources are unioned and conformed here.** | dbt |
  | `gold` | Contracted marts: `eligibility`, `member_months`, `medical_claim`, `pharmacy_claim` (Tuva core data model shape) and utilization marts | dbt |
  | `observation` | Pipeline health: `pipeline_event_log`, `expectations`, `feed_freshness`, `feed_volume`, `rescued_data`, `file_ledger`, `row_parity`, `seeded_defects`, `dq_scorecard` | dbt + pipelines |
  | `outbound` | What goes back to payers: `eligibility_reconciliation`, `return_file_<source_id>`, plus the `outbound` volume (return files land in the outbound S3 bucket). Tighter grants than `observation`. | dbt + the monthly job |

- A per-source bronze schema is created by the `source_landing` Terraform module, along with that source's volume.
  Grants per source are schema-level in bronze (for example, `source_ops_b` gets `source_b_bronze` only). Row filters on
  `source_id` handle the same isolation in silver and gold.
- Common columns everywhere: `source_id`, `_ingested_at`, `_source_file`.

## Environments: dev → preprod (`main`) → prod (`release`)

| Env | Git trigger | Catalog | dbt target | Bundle target | Who deploys |
|---|---|---|---|---|---|
| **dev** | Feature branch, from your laptop | `lab_dev` (schemas `dbt_<user>_*`; CI uses `ci_pr<N>_*`) | `dev` | `dev` (`mode: development`, triggers paused) | You, locally |
| **preprod** | Merge to **`main`** | `lab_preprod` | `preprod` | `preprod` (`mode: production`, file triggers **paused**) | GitHub Actions, `preprod-deployer` SP |
| **prod** | Merge `main` → **`release`** (PR, approval required) | `lab_prod` | `prod` | `prod` (`mode: production`, triggers live, `run_as` `prod-runner` SP) | GitHub Actions, `prod-deployer` SP via the `prod` GitHub environment |

- **Data flow.** Payer files land once, in prod (`lab_prod.landing`), and are ingested once into `lab_prod.<source_id>_bronze`.
  - dev and preprod dbt **read prod bronze** and write to their own catalog. Their service principals and groups get `SELECT` on the `lab_prod.*_bronze` schemas and nothing else in prod.
  - Ingestion code is tested in dev and preprod against **golden drops** in each catalog's own `landing` volume, so real files are never ingested twice.
- **Promotion is git, not clicks.**
  - **PR → main:** bundle validate, slim CI into `lab_dev.ci_pr<N>_*` (`state:modified+`, deferred to the prod manifest), and unit tests.
  - **Merge → main:** deploy preprod, a full `dbt build` and tests in `lab_preprod`, and the golden-drop integration test.
  - **PR main → release:** shows the preprod results. Merging it deploys prod.
  - **Hotfix:** branch from `release`, PR into `release`, then merge `release` back into `main`.
- **Branch protection.** `main` and `release` are protected: no direct pushes, CI must be green, and `release` needs approval.
  OIDC federation policies scope each service principal to its branch or environment: `preprod-deployer` → `ref:refs/heads/main`, and `prod-deployer` → `environment:prod`.
- **dbt naming.** `generate_schema_name`:
  - `dev` → `dbt_<user>_<schema>`
  - CI → `ci_pr<N>_<schema>`
  - `preprod` and `prod` → the custom schema name as is

  The catalog comes from the target.
- **Cost.** preprod builds run only on merge (never on a schedule) and its triggers stay paused. Never run `--target preprod|prod` locally.

## Pipeline observability (bronze must fail loudly, quarantine quietly, and prove both)

- **Expectations come from the registry** (`config/sources/<id>.yml → expectations:`), in three tiers:
  - `expect` (warn and keep)
  - `expect_or_drop` (the row goes to `<source_id>_bronze.<feed>_quarantine` tagged with the failing rule; nothing is silently dropped)
  - `expect_or_fail` (structural breakage: missing key columns, wrong delimiter)
- **Schema drift:** `cloudFiles.schemaEvolutionMode` is set per feed in the registry, `_rescued_data` is always kept,
  and expected columns are checked against the registry.
- **Event log** is published to `observation.pipeline_event_log`. dbt `observation` models (expectations, feed_volume,
  feed_freshness, rescued_data, file_ledger, row_parity) are the single place to read pipeline health.
- **Alerts, cheapest first:**
  1. Pipeline notifications (update/flow failure) and job `on_failure` + `health` rules.
  2. A post-ingest SQL check task that fails the job on breakage. It runs only when data arrives.
  3. Exactly one scheduled SQL Alert, run daily, for late or missing files.

  Don't add frequently scheduled alerts: every run wakes the warehouse.
- **Tests:**
  - Unit: pytest on `libs/`, local.
  - Golden-drop integration: deploy to dev, then assert **exact** passed/dropped/quarantined counts per rule from the event log.
  - dbt: source freshness, `warn_if`/`error_if`, `store_failures`, contracts.
  - Break-it drills in `tests/drills/`, each with an `expected_outcomes.yml` entry, run with `/break-it`.
- A drill **passes only if the expected outcome happened and nothing else did.** An alert that never fired
  is a failure even if the data looks fine.

## Commands

- Databricks CLI profiles: `acct` (account console) and `lab` (workspace). Add `-p lab` to workspace commands.
- Bundles: `databricks bundle validate|summary -t dev|preprod|prod` from inside `bundles/<name>/`. Only `-t dev`
  is deployed by hand; preprod and prod deploy from CI (`main` and `release`).
- dbt: from `bundles/transform/dbt`, `uv run dbt build --target dev --select <selector>`. Always use a scoped selector.
  Locally, use the `dev` target only.
- Terraform: from `terraform/<stack>`, `terraform init` / `plan` / `apply`. Destroy in reverse order, 40 → 10.
- Python: `uv sync`, `uv run pytest`.
- Spend: `/cost-check`.

## Skills (`.claude/skills/`)

| Skill | Use it to |
|---|---|
| `/add-payer-files` | Land a new drop for an existing source (mode A) or onboard a new source end to end (mode B) |
| `/phase-check <n>` | Check a plan phase's "done when" criteria and log the result to `docs/progress.md` |
| `/cost-check` | Total credits used against the $400 cap and apply the tripwires |
| `/tf-stack <stack>` | Run fmt → validate → plan, review the plan against the hard rules, then hand over `apply` |
| `/bundle-ship <bundle> <dev\|preprod\|prod>` | dev: validate, hand over deploy/run, verify. preprod/prod: walk the merge to `main` or the `main` → `release` PR, then verify the CI deploy |
| `/dbt-dev-loop <selector>` | Run a scoped dev build and tests, compare against prod, check contracts |
| `/break-it <drill>` | Run one breakage drill and verify exactly the expected outcome happened (state, counts, alert) |
| `/triage-run <job\|pipeline>` | Classify a failed run, walk the fix checklist, hand over `repair-run` |
| `/dq-scorecard` | Score dbt tests against the generator's seeded-defect manifest |
| `/teardown` | Destroy in order and verify nothing is left running or billing |

## MCP servers (`.mcp.json`)

- **dbt** (`uvx dbt-mcp`): runs the dbt CLI against `bundles/transform/dbt` with your `~/.dbt/profiles.yml`.
  Start Claude from the repo root so `${PWD}` resolves, and run `uv sync` first so `.venv/bin/dbt` exists.
  Mutating dbt tools (`build`, `run`, `test`) target `dev` only.
- **terraform** (`hashicorp/terraform-mcp-server`): public registry docs for the `databricks` and `aws`
  providers. It runs in Docker, so install Docker or Colima, or swap in the Go binary.
- **Databricks managed MCP**: add it in Phase 3 once the workspace exists. In the workspace, open
  Unity Gateway → MCPs, follow its Claude Code setup, then
  `claude mcp add --transport http databricks-sql https://<workspace-host>/ai-gateway/mcp-services/system.ai.dbsql`.
  The legacy endpoint is `/api/2.0/mcp/sql`. Keep the host out of git by using `--scope local`.
  Every query bills the SQL warehouse.

## Plan

The full curriculum (phases, data sources, file tree, cost allocation) lives in `docs/plan.md` once the
user commits it. Progress is logged in `docs/progress.md`.
