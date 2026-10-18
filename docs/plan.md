# Plan: `databricks-learning` — standalone multi-source ingestion lakehouse on Databricks (AWS us-west-2)

## Context

You want to learn Databricks hands-on and test it properly by building a realistic lab yourself: infrastructure, ingestion, storage, SQL, dbt, orchestration, jobs, and the CLI and UI for each. You'll write all the code. This plan is the curriculum and the target design.

**Reference model: Halcyon Health (fictional).** Halcyon is a value-based care company whose data team ingests eligibility, medical and pharmacy claims, prior auth and ADT from many payers, each with its own layout. Each source is standardized into one common model with dbt on Databricks/AWS, alongside reconciliation and return files. The lab simulates that with **public synthetic payer data**.

**Decisions so far:**
- **Standalone.** All code lives in your GitHub repo `databricks-learning`. Nothing plugs into existing infrastructure, repos or warehouses.
- **AWS region us-west-2.** This keeps the lab physically apart from any existing workloads. The lab gets its own state, buckets and IAM roles.
- **Hard budget: $400 total.** That's the size of the Databricks trial's credits, so the plan is designed to finish **inside the 14-day trial on credits alone**, with real spend near $0 (§8).
- **Serverless workspace** (`compute_mode = "SERVERLESS"`). There's no customer VPC, NAT gateway, EC2 or root bucket, so AWS-side cost is pennies of S3. Classic-workspace networking is study-only (§8).
- **Databricks 14-day trial** on AWS.
- **Three environments:** dev (feature branches) → preprod (`main`) → prod (`release`), promoted by git (§3 ADR 1, Phase 8).
- **About 2–3 weeks**, but all billable work fits the 14 trial days. Phase 0 happens before the trial clock starts.
- **How we work:** this session is your guide and reviewer. You run every command in another terminal tab and paste back output and errors. I review your Terraform, SQL and YAML. I don't scaffold lab code. The one exception is the Claude tooling in §9 (`CLAUDE.md` and the runbook skills), which I write on approval.

---

## 0. Guardrails (do first)

1. **Synthetic data only.** No real claims or PHI go in the trial workspace.
2. **Self-contained Terraform.** Every resource is created by this repo's stacks. Nothing reads or imports existing resources: no `data` sources pointing at existing VPCs, buckets or roles, and no shared state.
3. **Own state bucket.** Create `databricks-learning-tfstate-usw2` in `00-bootstrap`.
4. **Tag everything** through provider `default_tags`: `project=databricks-learning`, `owner=<you>`, `ttl=<date>`. Because nothing is shared, `terraform destroy` cleans up completely.
5. **Budget tripwires before anything billable:**
   - An **AWS Budget of $25**, alerting at 50, 80 and 100%.
   - A **Databricks account budget** (account console → Usage → Budgets) alerting at **$150, $250 and $325**.
   - **At $325, stop:** destroy everything and finish the rest of the work as study only.
   - Sign up for the trial the way that does **not** auto-convert to paid billing. After signup, check the account console → Subscription.
6. **Repo hygiene (public GitHub).** Commit an `.example` file and keep the real values out of git:
   - Gitignore `*.tfvars`, `.terraform/`, `*.tfstate*`, `.databrickscfg` and `profiles.yml`.
   - Never commit account IDs, ARNs, workspace URLs or tokens.
7. **AWS auth.** Use a profile with IAM and S3 rights. Pin `region = "us-west-2"` in every provider block, because your `~/.aws/config` defaults to us-east-1.

---

## 1. What the lab simulates

| Real-world need | Lab equivalent |
|---|---|
| Each source sends its own eligibility and claims layout | 3 core payer sources plus 2 stretch sources, each with a different schema, format and cadence |
| SFTP drop → raw zone | `aws s3 cp` into a landing S3 bucket (us-west-2), exposed as a Unity Catalog **external volume** |
| Ingestion and standardization | **Lakeflow (declarative) pipelines** using Auto Loader → **one bronze schema per source** (`<source_id>_bronze`), one pipeline per source, driven by a **feed registry** (YAML) |
| dbt modeling | `dbt-databricks`: sources **come together in `silver`** (per-source `stg_` → unioned and conformed `int_`) → `gold` marts (shaped like the Tuva data model) |
| Reconciliation and return files | A dbt mart that diffs eligibility and writes an outbound file to a volume |
| Data quality | dbt tests, unit tests and contracts, plus Lakeflow expectations, scored against a **seeded-defect manifest** |
| Per-customer isolation and PHI controls | UC grants, **row filters by source**, **column masks** on PHI-like fields |
| Orchestration | Lakeflow Jobs: file-arrival trigger → pipeline → dbt task → DQ SQL → alert |
| Platform as code | Terraform (AWS + account + UC), Asset Bundles (jobs, pipelines, dbt), CI with OIDC |

---

## 2. Data sources (public and free)

| Source | Dataset | Simulated feed | Cadence | Why it's useful |
|---|---|---|---|---|
| **A — "Medicare FFS"** | [CMS Synthetic Medicare Enrollment, FFS Claims & PDE (2023)](https://data.cms.gov/collection/synthetic-medicare-enrollment-fee-for-service-claims-and-prescription-drug-event): 8,671 beneficiaries, RIF layout | **Pipe-delimited `.txt`**; separate bene, inpatient, outpatient, carrier and Part D files | Monthly | Real CMS column names, header/line splits, Part D |
| **B — "Commercial (Aetna-style)"** | [Tuva `synthetic-data-generator`](https://github.com/tuva-health/synthetic-data-generator), Aetna layout (178-col medical, 71 pharmacy, 63 eligibility) | **CSV.gz**; convert eligibility to **fixed-width** | Weekly | A wide, messy commercial layout |
| **C — "Regional plan (HCCI-style)"** | Same generator, HCCI 2.0 layout (enrollment plus separate IP, OP, physician and pharmacy files) | Convert to **Parquet** | Daily | Claims split by setting, a different grain |
| *D — stretch: FHIR* | [Blue Button 2.0 synthetic bulk](https://bfd-public-test-data.s3.amazonaws.com/data-synthetic/SyntheticDataApiResponses/) (Patient / Coverage / EOB NDJSON). Check that the bucket is still live. | **Nested JSON** | Daily | Semi-structured flattening (`explode`, `from_json`, VARIANT) |
| *E — stretch: EDI / provider* | [Databricks x12-edi-parser](https://github.com/databricks-industry-solutions/x12-edi-parser) samples (837/835/834), or Synthea encounters as an ADT-like feed | **X12 / CSV** | Daily | Parsing a hierarchical EDI format |

- **Seeded defects:** the Tuva generator's default `connector_eval` profile injects defects (bad codes, overlapping coverage, reversals and voids) and writes an `issue_manifest.csv`. Your dbt tests get a measurable target: catch rate per defect class.
- **Common target model:** the [Tuva core data model](https://thetuvaproject.com/core-data-model/overview) (`eligibility`, `medical_claim`, `pharmacy_claim`) is the contract your marts conform to. You write all the SQL; Tuva is only the spec.

---

## 3. Proposed monorepo file tree (`databricks-learning/`)

```
databricks-learning/
├── README.md                      # architecture diagram, how to run, rebuild-from-zero steps
├── CLAUDE.md                      # rules, naming, working agreement (§9a)
├── .mcp.json                      # dbt MCP, Databricks managed MCP, Terraform MCP (§9c) — no secrets
├── .claude/
│   ├── settings.json              # read-only allowlist / mutating denylist (§9d)
│   └── skills/                    # add-payer-files, phase-check, cost-check, tf-stack, bundle-ship,
│                                  #   dbt-dev-loop, triage-run, dq-scorecard, teardown (§9b)
├── Makefile                       # thin wrappers: make tf-plan STACK=30, make deploy T=dev, make drop SRC=a
├── pyproject.toml / uv.lock       # uv workspace root (members: generators, libs/source_ingest; groups: dev, dbt)
├── .python-version                # pinned interpreter for uv
├── .pre-commit-config.yaml        # terraform fmt, tflint, sqlfluff (databricks dialect), ruff, yamllint
├── .github/workflows/
│   ├── terraform.yml              # fmt/validate/plan on PR; apply on main (manual-approval environment)
│   ├── pr.yml                     # PR → main: lint, unit tests, tf plan, bundle validate, dbt slim CI into ci_pr<N>_*
│   ├── pr-integration.yml         # label `run-integration`: golden-drop test on a per-PR dev deploy
│   ├── pr-cleanup.yml             # PR closed: drop ci_pr<N>_* schemas, destroy per-PR deploys
│   ├── preprod.yml                # push to main: bundle deploy -t preprod, full dbt build in lab_preprod, golden-drop test
│   ├── prod.yml                   # push to release: deploy -t prod (env `prod`, approval), smoke run, release tag
│   ├── drift.yml                  # weekly terraform plan -detailed-exitcode on every stack
│   └── rollback.yml               # workflow_dispatch: redeploy a previous release tag to prod
│
├── terraform/
│   ├── 00-bootstrap/              # us-west-2 state bucket (versioned, native S3 locking), $25 AWS budget,
│   │                              #   GitHub OIDC → AWS role, default_tags
│   ├── 10-workspace/              # databricks_mws_workspaces with compute_mode = "SERVERLESS" (no VPC,
│   │                              #   no cross-account role, no root bucket), account budget + alerts
│   ├── 20-account/                # metastore (us-west-2) + assignment, account groups (data_engineers,
│   │                              #   analysts, source_ops), service principals (preprod-deployer, prod-deployer, prod-runner)
│   ├── 30-unity-catalog/          # landing + outbound buckets, UC IAM role (self-assuming trust),
│   │                              #   storage credential, external locations, catalogs, schemas, volumes, grants
│   ├── 40-workspace-config/       # serverless SQL warehouse (2X-Small, auto-stop 5m), serverless budget
│   │                              #   policy (tags usage), secret scope
│   └── modules/
│       ├── source_landing/        # one call per source: S3 prefix + external volume + grants
│       ├── uc_env_catalog/        # catalog + standard schemas for an env
│       └── service_principal/
│
├── generators/                    # local Python only; produces the source "drops"
│   ├── source_a_cms_rif/          # download + unzip + slice into monthly files
│   ├── source_b_tuva_aetna/       # run generator, convert eligibility to fixed-width
│   ├── source_c_tuva_hcci/        # run generator, write Parquet
│   ├── source_d_bluebutton/       # (stretch)
│   ├── source_e_x12/              # (stretch)
│   └── drop_simulator.py          # replays files into s3://…/landing/<source>/<feed>/<yyyy>/<mm>/<dd>/
│
├── libs/source_ingest/            # small shared Python package, built as a wheel by bundles
│   ├── registry.py                # loads config/sources/*.yml
│   ├── parsers.py                 # fixed-width, FHIR, X12 helpers
│   └── tests/
│
├── config/sources/                # THE onboarding contract: one YAML per source
│   ├── source_a.yml               # feeds, format, delimiter, path glob, cadence, expected cols, PK, PHI cols
│   ├── source_b.yml
│   └── source_c.yml
│
├── bundles/
│   ├── ingestion/                 # Asset Bundle #1 (Python)
│   │   ├── databricks.yml         # targets: dev (development), preprod (main), prod (release, run_as SP)
│   │   ├── resources/{bronze_source_a.yml, bronze_source_b.yml, …, ingest_jobs.yml}  # one pipeline per source → <source_id>_bronze
│   │   ├── src/pipelines/{bronze_from_registry.py, expectations.py, quarantine.py}
│   │   ├── src/checks/post_ingest_checks.sql              # SQL task: fail job on breakage
│   │   └── src/notebooks/explore_*.py
│   │
│   ├── transform/                 # Asset Bundle #2 (dbt-sql template)
│   │   ├── databricks.yml
│   │   ├── resources/dbt_job.yml
│   │   └── dbt/
│   │       ├── dbt_project.yml / packages.yml / profiles.yml (template, no secrets)
│   │       ├── seeds/             # source_registry, code sets (ICD-10 ranges, NDC/J-codes, POS, rev codes)
│   │       ├── snapshots/         # member eligibility SCD2
│   │       ├── models/
│   │       │   ├── staging/source_a|b|c/  # → silver: ONLY source-specific SQL (rename, cast, unpivot), reads <source_id>_bronze
│   │       │   ├── intermediate/          # → silver (sources unioned here): int_eligibility_spans, int_medical_claim_unioned,
│   │       │   │                          #   int_claim_adjustments, int_member_xwalk
│   │       │   └── marts/
│   │       │       ├── core/              # eligibility, member_months, medical_claim, pharmacy_claim (contracted)
│   │       │       ├── utilization/       # ed_visits, ip_admits, readmits_30d, pmpm
│   │       │       ├── observation/       # pipeline health: expectations, feed_volume, feed_freshness, rescued_data,
│   │       │       │                      #   file_ledger, row_parity, seeded_defects, dq_scorecard
│   │       │       └── outbound/          # eligibility_reconciliation, return_file_<source> → outbound volume
│   │       ├── macros/            # union_sources(), generate_schema_name
│   │       └── tests/ + unit tests (dbt ≥1.8)
│   │
│   └── ops/                       # Asset Bundle #3 (optional): dashboards (.lvdash.json), alerts, Genie space
│
├── tests/
│   ├── unit/                      # pytest: parsers, registry → expectation builder (local, free)
│   ├── integration/               # golden-drop pipeline test: asserts exact expectation counts from event log
│   ├── fixtures/<source_id>/      # golden drops with known good/bad row counts per rule
│   └── drills/                    # break-it fixtures + expected_outcomes.yml (see Phase 4b)
│
└── docs/
    ├── architecture.md
    ├── onboarding-a-new-source.md # runbook: YAML → drop → staging model → done
    ├── data-contracts.md
    ├── adr/                       # 0001-catalog-per-env, 0002-bronze-lakeflow-silver-dbt, …
    ├── ui-vs-cli-vs-code.md       # your cheat sheet as you go
    └── evaluation.md              # your findings notebook (§6)
```

### Python toolchain: uv (everything Python goes through it)

- **One uv workspace** at the repo root. The root `pyproject.toml` declares `[tool.uv.workspace] members = ["generators", "libs/source_ingest"]`. `uv.lock` is committed, and `.python-version` pins the interpreter (pick one the Databricks serverless environment supports).
- **Dependency groups** in the root `pyproject.toml`:
  - `dev`: ruff, pytest, pre-commit, sqlfluff + sqlfluff-templater-dbt, yamllint
  - `dbt`: dbt-core + dbt-databricks, pinned
  - `spark` (optional): databricks-connect, only for tests that need Spark
- **Every command is `uv run …`:** `uv run pytest`, `uv run dbt build`, `uv run ruff check`, `uv run python generators/drop_simulator.py`. One-off tools run with `uvx` (`uvx dbt-mcp`).
- **The shared library is a wheel.** `libs/source_ingest` is built with `uv build --wheel`. Each bundle declares it under `artifacts:` (`build: uv build --wheel`) and lists the built wheel in its serverless environment `dependencies`, so pipelines and jobs run exactly the locked code.
- **CI uses the same toolchain:** `astral-sh/setup-uv` with caching, then `uv sync --locked`. CI fails if `uv.lock` is out of date. Nothing is `pip install`ed by hand anywhere.

**Design decisions (write each as an ADR):**
1. **Three environments, one catalog each:** **dev** (`lab_dev`, feature branches, `dbt_<you>_*` schemas) → **preprod** (`lab_preprod`, deployed on merge to **`main`**) → **prod** (`lab_prod`, deployed on merge `main` → **`release`**). Schemas: `landing`, `<source_id>_bronze` (one per source), `silver`, `gold`, `observation` (pipeline health), `outbound` (reconciliation and return files for payers, with tighter grants). Payer files land and are ingested **once**, into prod bronze. dev and preprod dbt read the `lab_prod.*_bronze` schemas (`SELECT` only) and write to their own catalog. Ingestion code is tested in dev and preprod against golden drops in each catalog's own `landing` volume.
2. **Bronze in Lakeflow, everything after in dbt.** Ingestion is a streaming and schema-evolution problem; modeling is a SQL problem.
3. **Bronze per source, union in silver.** Each source gets its own `<source_id>_bronze` schema and its own pipeline, running the same generic code parameterized by `source_id`. A broken payer can't stall the others, and grants, lineage and teardown are per source. Sources first meet in `silver`. Onboarding means a registry YAML file, one pipeline resource entry, a Terraform module call and a staging folder; no new pipeline code.
4. **Multiple bundles in one repo** (ingestion, transform, ops). Each deploys independently, and shared code is a wheel built from `libs/`. This is how a monorepo scales to many domains.
5. **Isolation through UC row filters** on `source_id`, not by copying tables.

---

## 4. Phased build (Phase 0 pre-trial, then Phases 1–9 in 13 trial days, plus 1 buffer day)

> **Trial clock:** the trial is about $400 of credits over 14 days. Do Phase 0 **before** signing up. At the end, either convert to pay-as-you-go or `terraform destroy`.
>
> **Loop for each component:** UI first (learn) → CLI (inspect and automate) → code (Terraform, bundle or dbt) → destroy and redeploy from code.

### Phase 0 — Pre-trial prep (Days 0–1, no trial clock)
- Install the CLI tools: `brew tap databricks/tap && brew install databricks`, `terraform`, `uv`.
- Set up the uv workspace: root `pyproject.toml` with the workspace members and the `dev`/`dbt` groups, `.python-version`, then `uv sync` and a committed `uv.lock`. dbt-databricks gets installed through the `dbt` group, not globally.
- Start CI early: a `pr.yml` with only ruff, pytest, terraform fmt/validate and `uv sync --locked`. It's free and grows each phase (see Phase 8).
- Work through the §0 guardrails: `.gitignore`, the AWS profile, and `default_tags`.
- Run the generators locally for sources A, B and C. Study the layouts and write `config/sources/*.yml` by hand.
- Write `terraform/00-bootstrap` and apply it in us-west-2: state bucket, budget alarm.
- Read:
  - The UC object model.
  - Lakeflow pipelines concepts.
  - The [bundle-examples](https://github.com/databricks/bundle-examples) folders `dbt_sql` and `lakeflow_pipelines_python`, plus `knowledge_base/` → `share_files_across_bundles` and `job_file_arrival`.
- **Done when:** the three source folders are on disk, there are 3 registry YAMLs, and the state bucket exists in us-west-2.

### Phase 1 — Account and workspace with Terraform (Days 2–3; sidebar: *Compute, Catalog*)
- Sign up for the trial and choose **us-west-2** for the workspace.
- In the **account console UI**: find the account ID and see whether a metastore was auto-created (and in which region). Then create an account-level service principal with an OAuth secret for Terraform.
- CLI:
  - `databricks auth login --host https://accounts.cloud.databricks.com --account-id <id> --profile acct`
  - `databricks account workspaces list -p acct`
- Code:
  - `10-workspace`: `databricks_mws_workspaces` with `compute_mode = "SERVERLESS"` and `aws_region = "us-west-2"`, plus the account budget and its alerts.
  - `20-account`: metastore assignment, groups, service principals.
- Study only, not applied: read the classic `HYBRID` path in [terraform-databricks-sra/aws](https://github.com/databricks/terraform-databricks-sra/tree/main/aws) (cross-account role, root bucket, customer VPC). Optionally run `terraform plan` without applying, so you know what a classic deployment costs and involves without paying for NAT and EC2.
- Exercise: create a group in the UI, `terraform import` it, then remove the click-ops version. This teaches drift and import.
- **Done when:** `terraform apply` creates a workspace you can log into, and both `databricks auth login --host <workspace-url> --profile lab` and `databricks current-user me -p lab` work.

### Phase 2 — Unity Catalog storage and governance (Days 3–4; sidebar: *Catalog*)
- UI: in Catalog Explorer, create a throwaway catalog, schema and managed volume, then delete them.
- Code (`30-unity-catalog`):
  - Landing and outbound buckets.
  - The UC IAM role with a self-assuming trust policy.
  - The storage credential and external locations.
  - The `lab_dev`, `lab_preprod` and `lab_prod` catalogs with their schemas, using a `uc_env_catalog` module called three times. A golden-drop `landing` volume goes in dev and preprod.
  - Cross-env grant: the dev/preprod groups and service principals get `SELECT` on the `lab_prod.*_bronze` schemas only.
  - Each `source_landing` module call creates that source's volume **and** its `<source_id>_bronze` schema, with schema-level grants (for example, `source_ops_b` gets `source_b_bronze`).
  - **One external volume per source** (`lab_prod.landing.source_a` and so on).
  - Group grants.
- CLI: `databricks catalogs list`, `schemas list lab_prod`, `volumes list lab_prod landing`, `grants get catalog lab_prod`.
- **Done when:** after `python generators/drop_simulator.py --source a --date 2026-01-31` and an `aws s3 cp` to the landing bucket, the files appear under `/Volumes/lab_prod/landing/source_a/...`, both in the UI and in `databricks fs ls dbfs:/Volumes/lab_prod/landing/source_a`.

### Phase 3 — Explore with SQL and notebooks (Day 5; sidebar: *SQL Editor, Queries, Workspace, Data Ingestion, Visual Data Prep*)
- In `40-workspace-config`, create a serverless SQL warehouse (2X-Small, auto-stop after 5 min) and a serverless budget policy.
- **SQL Editor:** profile every source with `read_files('/Volumes/...', format=>'csv', sep=>'|', header=>true)`. Save the results as a "Feed profiling" set in **Queries**.
- **Data Ingestion UI:** upload one code-set seed. **Visual Data Prep:** try it once on a source B file.
- In a serverless notebook, do the same profiling in PySpark: `_metadata.file_path`, explicit schema vs inferred, and the rescued data column.
- **Done when:** you can describe each source's quirks from your saved queries.

### Phase 4 — Bronze ingestion with a Lakeflow pipeline and bundle (Days 6–7; sidebar: *Jobs & Pipelines, Runs*)
- UI first: build a one-source pipeline in the Lakeflow Pipelines editor (Auto Loader → streaming table). Explore the DAG, event log and expectations.
- Then as code:
  - Run `databricks bundle init` (pick the Python/Lakeflow template) to create `bundles/ingestion`.
  - Write `bronze_from_registry.py`: it reads the `source_id` pipeline parameter, loads `config/sources/<source_id>.yml`, and declares one streaming table per feed in `<source_id>_bronze` with `_ingested_at`, `_source_file`, `_rescued_data` and schema evolution.
- CLI loop: `bundle validate -t dev` → `bundle deploy -t dev` → `bundle run -t dev bronze_source_a` → `pipelines list-pipelines` → `bundle summary`.
- Job: `ingest_job.yml` with a **file-arrival trigger** on each source volume.
- Exercise: drop one file with a new column (schema drift) and one with corrupt rows.
- **Done when:** dropping a new day of source C Parquet updates `lab_prod.source_c_bronze.*` with no manual steps, and the other sources' pipelines don't run.

### Phase 4b — Pipeline observability, alerting and break-it drills (Day 8)

The streaming layer needs to **fail loudly, quarantine quietly, and prove both with tests**. The design has five layers, each with a test that shows it works.

**1. In-pipeline expectations** (in `bronze_from_registry.py`, driven by the registry)
- Rules per feed come from `config/sources/<id>.yml → expectations:` and fall into three tiers:
  - `expect` (warn and keep): soft rules, e.g. a non-standard plan code.
  - `expect_or_drop` with quarantine: bad rows, e.g. null member_id or a service date in the future.
  - `expect_or_fail`: structural breakage, e.g. zero expected columns present or the wrong delimiter (a single-column read).
- **Quarantine instead of silent drops:** for each `expect_or_drop` rule, a sibling streaming table `<source_id>_bronze.<feed>_quarantine` catches the inverse predicate and records which rule failed. Nothing disappears without a trace.
- **Schema drift:** set `cloudFiles.schemaEvolutionMode` per feed (`addNewColumns` for sources you trust, `rescue` for messy ones), always keep `_rescued_data`, and run a `schemaHints` check against the registry's `expected_columns`.

**2. Observability tables** (UC, queryable)
- Publish the pipeline **event log to a UC table** (`observation.pipeline_event_log`, set in the pipeline settings).
- dbt `observation` models on top:
  - `expectations`: per update, flow and rule, the passed, failed and dropped counts from `details:flow_progress.data_quality`.
  - `feed_volume`: rows per source, feed and drop, compared with the trailing 4-drop average. Flags beyond ±30% and empty drops.
  - `feed_freshness`: last `_ingested_at` per source and feed against the registry cadence SLA, with `late` and `missing` flags.
  - `rescued_data`: count of rows with non-null `_rescued_data` per file.
  - `file_ledger`: every file name, size and hash seen. Flags duplicate re-drops and out-of-order dates.
  - `row_parity`: bronze rows vs `read_files` count per file.

**3. Alerts** (cheapest first, because every scheduled check wakes the warehouse)
- **Pipeline notifications** in each `bronze_<source_id>.yml`: email on `on-update-failure`, `on-update-fatal-failure` and `on-flow-failure`.
- **Job-level:** `email_notifications.on_failure`, plus `health` rules (`RUN_DURATION_SECONDS`, and streaming backlog `STREAMING_BACKLOG_FILES/SECONDS`) with `on_duration_warning_threshold_exceeded` / `on_streaming_backlog_exceeded`.
- **Post-ingest SQL check task** in `source_daily`: right after the pipeline update, a SQL task asserts there are no `expect_or_fail` trips, quarantine stays under N% of the drop, rescued rows = 0, the volume is within band and row parity holds. **Failing the task fails the job, which sends the notification.** This runs only when data arrives, so it costs nothing when idle.
- **Scheduled SQL Alerts** only for things an event can't catch, i.e. **late or missing files**: one `feed_freshness` alert, scheduled **once a day**. That keeps the cost down.
- **Destination:** a `databricks_notification_destination` (email; optionally a personal Slack or Teams webhook), managed in `terraform/40-workspace-config`.
- *Stretch:* UC **data quality monitoring** (anomaly detection) on `gold`. Run `/cost-check` first.

**4. Tests at three levels**
- **Unit (local, free):** pytest for `libs/source_ingest` parsers and the registry → expectation builder (fixed-width slicing, FHIR flattening, a rule-to-SQL predicate). Use Databricks Connect (serverless) only where Spark is required.
- **Pipeline integration (dev target):** a **golden drop** per source in `tests/fixtures/<source_id>/` with known counts: N good rows plus a known number violating each rule. Deploy the bundle to `dev`, point it at the test volume, and run one update. A pytest then queries `lab_dev.observation.pipeline_event_log` and asserts the **exact** expected passed, dropped and quarantined counts per rule. This is "testing on expected".
- **dbt:**
  - `dbt source freshness` with `loaded_at_field: _ingested_at` and `warn_after` / `error_after` per cadence.
  - Volume and accepted-range tests with `warn_if` / `error_if`.
  - `store_failures: true` into `lab_dev.dbt_test_failures`.
  - Contracts on the marts.

**5. Break-it drills** (`tests/drills/`, run with `/break-it`). Each drill is a fixture file plus an **expected outcome**. You drop it and Claude verifies that exactly that outcome happened:

| Drill | Fixture | Expected outcome |
|---|---|---|
| `new_column` | Extra column in the source B eligibility file | `addNewColumns`: the stream restarts with the new column, `expectations` is clean, and the change is noted in evaluation.md. `rescue`: the column lands in `_rescued_data` and the rescued check fails the job → email. |
| `missing_column` | `member_id` column removed | `expect_or_fail` trips → pipeline update fails → pipeline failure email |
| `wrong_delimiter` | Source A file comma-delimited instead of pipe | The single-column read trips `expect_or_fail` → fails loudly |
| `bad_rows` | 5% null member_id, 2% future service dates | Rows land in `_quarantine` with the rule name; the job passes if under the threshold and fails above it |
| `empty_file` | 0-byte or header-only drop | The volume check flags `empty_drop` → the SQL task fails → email |
| `duplicate_file` | The same file re-dropped under a new name | `file_ledger` hash match → flagged, and bronze row parity shows no double count (dedupe downstream) |
| `volume_spike` | 3× the normal row count | Volume band breach → job fails → email |
| `late_file` | Skip a cadence for source C | The daily freshness Alert fires the next morning |
| `corrupt_gzip` | A truncated `.csv.gz` | The pipeline flow fails → `on-flow-failure` email; check whether the rest of the update continues |
| `backfill_out_of_order` | An older date dropped after newer ones | Ingested; the ledger flags it as out of order; incremental dbt models still produce correct spans |

- **Done when:**
  - Every drill produces its expected outcome, and each result is logged in `docs/progress.md`.
  - The golden-drop integration test passes with exact counts.
  - At least one real email alert has been received for a pipeline failure, a SQL check task failure, and a freshness Alert.

### Phase 5 — dbt on Databricks (Days 9–11)
- Run `databricks bundle init dbt-sql` to create `bundles/transform`. Run dbt locally against the warehouse, with `target: dev` writing to `lab_dev`.
- What you write, in order:
  1. **Staging, one folder per source:** rename and cast to a shared vocabulary. This includes CMS RIF wide→long diagnosis columns, the Aetna 178-column layout, and HCCI's split-by-setting files.
  2. **Intermediate:** a `union_sources()` macro; **eligibility spans** (gaps-and-islands with overlap resolution); **claim adjustments** (reversals, voids, latest version wins); a member crosswalk.
  3. **Marts:** contracted `eligibility`, `member_months`, `medical_claim` and `pharmacy_claim`; utilization marts; `outbound.eligibility_reconciliation`, `outbound.return_file_source_b`, and `observation.dq_scorecard` (test failures vs `issue_manifest.csv`).
- dbt-databricks features to practice:
  - `incremental_strategy='merge'` vs `replace_where`
  - `liquid_clustered_by`
  - three-level naming
  - `persist_docs` into UC comments
  - snapshots
  - one **Python model** (fixed-width or FHIR parsing)
  - unit tests on the span and adjustment logic
- Deploy: in `resources/dbt_job.yml`, define a dbt task (`dbt deps`, `dbt build --select tag:source_b+`), then `bundle deploy -t dev` and `bundle run`.
- **Done when:**
  - One `dbt build` produces the same `gold` shapes for all 3 sources.
  - Contracts and tests pass.
  - The DQ scorecard shows the defect catch rate.

### Phase 6 — Governance (Day 12; sidebar: *Catalog → lineage, permissions*)
- Add a row filter on `source_id` so `source_ops_b` sees only source B, and column masks on name, DOB and member ID for `analysts`.
- Add `phi=true` tags on those columns.
- Trace lineage from the volume all the way to `pmpm`.
- Query the system tables: `system.access.audit` (who read masked columns) and `system.billing.usage` (what the lab costs). Save both as queries.
- **Done when:** a restricted principal sees filtered or masked rows.

### Phase 7 — Orchestration end to end (Day 13; sidebar: *Jobs & Pipelines, Runs, Alerts*)
- Build a `source_daily` job: file arrival → pipeline update → dbt task → SQL DQ task → condition task → notification. Exercise job parameters (`source_id`), retries, a deliberate failure, and **repair run**.
- Build a monthly job that builds the `outbound` models and writes return files to the outbound volume. Add a check that fails if a payer's expected return file is missing; an empty `outbound` is a business failure, not a quiet success.
- Add a SQL **Alert** on `observation.feed_freshness`.

### Phase 8 — CI/CD for Databricks, done properly (built incrementally, finished Day 14)

CI/CD isn't a single day at the end. Each phase adds the checks for what it introduced, and Day 14 finishes the deploy and promotion workflows.

| Phase | CI gains |
|---|---|
| 0 | `pr.yml`: `uv sync --locked`, ruff, pytest (unit), terraform fmt/validate/tflint, gitleaks |
| 1–2 | Terraform plan on PR, posted as a PR comment (OIDC to AWS and Databricks) |
| 4 | `databricks bundle validate -t dev/preprod/prod` for every bundle |
| 4b | Unit tests for parsers and the expectation builder. The golden-drop integration test is runnable by label. |
| 5 | sqlfluff (dbt templater), dbt parse/compile, dbt slim CI into `lab_dev.ci_pr<N>_*` |
| 8 | Deploy workflows: preprod on `main`, prod on `release`. Cleanup, drift and rollback workflows. |

**Workflows (`.github/workflows/`):**
- **`pr.yml` (PR → `main`, no deploy):**
  - Lint: ruff, sqlfluff, yamllint, terraform fmt, tflint.
  - Run `uv run pytest tests/unit` and `bundle validate` for all targets.
  - Post the Terraform plan as a comment.
  - **dbt slim CI:** `dbt build --select state:modified+ --defer --state <prod manifest>` into `ci_pr<N>_*`. The prod manifest is the artifact uploaded by the last prod deploy.
  - Required checks gate the merge.
- **`pr-integration` (label-triggered):** deploys the ingestion bundle to `dev` with a per-PR name prefix and runs the golden-drop integration test. It costs DBUs, so it runs only when the `run-integration` label is added.
- **`preprod.yml` (push to `main`):**
  - Deploys every bundle with `-t preprod` as `preprod-deployer`.
  - Runs the full `dbt build` + `dbt source freshness` in `lab_preprod`, then the golden-drop integration test against `lab_preprod`.
  - Publishes the preprod manifest and results as artifacts.
- **`prod.yml` (push to `release`):**
  - `environment: prod` with required approval.
  - Deploys `-t prod` as `prod-deployer`, then runs a **smoke job** (one tiny dbt selector, plus checking that the pipeline is healthy and the triggers are live).
  - Uploads the prod `manifest.json` for future slim CI.
  - Creates a release tag (`vYYYY.MM.DD-N`) with generated notes.
- **`pr-cleanup.yml` (PR closed):** drops the `lab_dev.ci_pr<N>_*` schemas and destroys any per-PR dev bundle deployment.
- **`drift.yml` (weekly, `workflow_dispatch`):** `terraform plan -detailed-exitcode` on every stack. It opens an issue on drift. It calls only APIs and uses no Databricks compute.
- **`rollback.yml` (`workflow_dispatch` with a tag input):** checks out a previous release tag and redeploys `-t prod`. A rollback is a deploy of known-good code, never manual clicks.

**Practices to apply:**
- **No secrets:** GitHub OIDC → Databricks workload identity federation (one policy per service principal: `preprod-deployer` → `ref:refs/heads/main`, `prod-deployer` → `environment:prod`), and GitHub OIDC → an AWS IAM role for Terraform.
- **Pinned everything:** action SHAs, the `databricks/setup-cli` version, `uv.lock`, the dbt-databricks version, and Terraform provider lock files.
- **One deploy at a time:** a `concurrency:` group per environment, with `cancel-in-progress: false` for deploys. Bundles also take their own deployment lock.
- **Bundle settings:** `mode: production` for preprod and prod. In prod, `run_as` is `prod-runner`. Add top-level `permissions`, per-target `variables` (catalog, warehouse, notification email) and `presets` (tags, trigger pause status). Deploys carry git metadata (branch, SHA), so every job shows which commit is running.
- **Same artifact everywhere:** the wheel is built once per workflow run with `uv build`, and the same commit is promoted from `main` to `release`. Nothing is rebuilt differently for prod.
- **Test pyramid by cost:** unit tests and validation on every PR (free), then integration by label, a full build on preprod, and only a smoke run in prod.
- **Timeouts and budgets:** every job sets `timeout_minutes`, and CI jobs use serverless with the lab budget policy, so CI usage shows up in `/cost-check`.
- **Repo rules:** protect `main` and `release` (required checks, no force-push, linear history), and use CODEOWNERS for `terraform/` and `bundles/*/databricks.yml`. Pre-commit runs the same linters locally (`uv run pre-commit install`).
- **dbt naming:** targets `dev`, `ci`, `preprod` and `prod`. `generate_schema_name` gives dev `dbt_<user>_*`, CI `ci_pr<N>_*`, and preprod/prod the plain schema names.

**Done when:**
- A PR that changes one staging model passes all required checks, including slim CI, and leaves no `ci_pr<N>` schemas after it closes.
- Merging to `main` builds `lab_preprod` green.
- The `main` → `release` PR (approved) deploys to `lab_prod`, the smoke run passes, and a release tag is created.
- `rollback.yml` redeploys the previous tag successfully.
- One hotfix round-trip leaves `main` and `release` in sync.
- `drift.yml` reports no drift.

### Phase 9 — Consumption, evaluation, teardown (Day 15; sidebar: *Dashboards, Genie Agents, Query History, Playground*)
- Build an AI/BI **Dashboard** (lives by source, member months, PMPM, ED and IP per 1,000, feed freshness) and export it to `bundles/ops`.
- Build a **Genie space** over `gold` with instructions and example SQL.
- In **Query History** and the query profile, find the slowest dbt model and fix it.
- *Stretch (AI/ML sidebar), only if usage is under $250:* a few `ai_query()` calls in SQL, and an MLflow experiment logged from a serverless notebook. **Skip Model Serving endpoints**; browse the Serving UI only.
- Run `terraform destroy` (stacks 40 → 10), then rebuild from zero by following the README, and time it.

---

## 5. Sidebar coverage map (from your screenshot)

| Sidebar item | Phase |
|---|---|
| Workspace, Recents | 3 |
| Catalog | 2, 6 |
| Jobs & Pipelines, Runs | 4, 7 |
| Compute, SQL Warehouses | 1, 3 |
| Discover, Marketplace | 3 (browse only) |
| SQL Editor, Queries, Query History | 3, 9 |
| Dashboards, Genie Agents, Alerts | 7, 9 |
| Data Ingestion, Visual Data Prep | 3 |
| Playground, Agents, AI Gateway, Experiments, Features, Models, Serving | 9 (stretch) |

## 6. Findings notebook (`docs/evaluation.md`, filled in as you go)

A plain record of what you learn about Databricks itself, so you have evidence when the team discusses it.

| Area | What to record |
|---|---|
| Cost | DBUs per pipeline update and per dbt run, from `system.billing.usage` |
| dbt | Dialect and adapter quirks; how incremental strategies and macros behave |
| Ingestion | How Auto Loader handles schema drift, corrupt rows and file-arrival latency |
| Governance | How easy row filters, masks, audit and lineage were to set up; what the HIPAA path requires (compliance security profile, BAA) |
| IaC and DX | Pain points in the Terraform provider; how bundle deploys feel; CLI ergonomics |
| Orchestration | What Lakeflow Jobs can and can't do (triggers, repair, conditions) |
| AI/BI | How good Genie's answers are; dashboard authoring experience |

## 7. CLI cheat sheet to grow (`docs/ui-vs-cli-vs-code.md`)

| Area | Commands |
|---|---|
| Auth | `auth login`, `auth profiles`, `current-user me` |
| Unity Catalog | `catalogs`, `schemas`, `volumes`, `grants` with `list` / `get` |
| Files | `fs ls`, `fs cp` on `dbfs:/Volumes/...` |
| Bundles | `bundle init`, `validate`, `deploy`, `run`, `summary`, `destroy -t <target>` |
| Jobs | `jobs list`, `get-run`, `repair-run` |
| Pipelines | `pipelines list-pipelines`, `start-update` |
| Warehouses | `warehouses list` |
| Raw REST | `api get /api/2.1/...` |

Run `databricks --version` first. The CLI is now 1.x, and Asset Bundles were renamed **Declarative Automation Bundles** in March 2026, so docs may use either name.

## 8. Cost guardrails

**Hard cap: $400.** Target: **≤ $300 of trial credits used, about $0–5 of real AWS spend, $0 out of pocket.**

**Rough allocation** (estimates; check actuals daily):

| Bucket | Budget | How to stay under |
|---|---|---|
| SQL warehouse (dbt, SQL Editor, dashboards, Genie) | ~$120 | 2X-Small serverless, **auto-stop 5 min**. Develop dbt with `--select` on one source, not full builds. |
| Lakeflow pipelines (bronze) | ~$60 | Serverless, **triggered** mode, never continuous. Run only on a new drop. |
| Serverless jobs and notebooks | ~$50 | Standard (not performance-optimized) serverless mode. Detach idle notebooks. |
| CI + preprod builds | ~$40 | PR slim CI builds only `state:modified+`. Preprod builds once per merge to `main`, never on a schedule, with its triggers paused. |
| AI/ML stretch | ~$20 | Only if under $250 by Phase 9. No serving endpoints. |
| Buffer | ~$110 | Stop at the $325 tripwire. |
| AWS (S3 landing/outbound/state, IAM) | <$5 | Serverless workspace means no NAT, EC2 or VPC. Data is MB to low GB. |

**Rules:**
- **Keep the data small.** Use the CMS 2023 synthetic set (8.7K beneficiaries) and Tuva at **2,000–5,000 members per source** instead of the 10K default. Skip DE-SynPUF's full scale. Synthea, if you do it, at 5K.
- **No classic clusters or all-purpose compute.** Everything is serverless, so nothing idles for long.
- **No continuous pipelines, no streaming jobs left running, no serving endpoints.**
- **Track daily:** `SELECT usage_date, sku_name, SUM(usage_quantity) FROM system.billing.usage GROUP BY ALL ORDER BY 1 DESC`. Look at it in the account console too. Record the numbers in `evaluation.md`, since cost per pipeline or dbt run is itself a finding.
- **Finish inside the 14 days.** Phase 0 is pre-trial, Phases 1–9 take 13 days, and the last day is a buffer. Do **not** convert to pay-as-you-go.
- **Teardown on the last day:** `terraform destroy` stacks 40 → 10. The tag query in Verification should then return nothing.

## 9. Claude tooling for the repo: CLAUDE.md, skills, MCP (I write these on approval)

**Working agreement for all of it:** Claude **reads and verifies**. You **write code and run anything that changes state**. Skills and settings enforce that split:
- Claude may run read-only commands itself: `databricks … list|get`, `bundle validate|summary`, `terraform plan|validate`, `aws s3 ls`, and SELECT queries.
- Claude prints mutating commands for you to run in your other tab: `bundle deploy|run|destroy`, `terraform apply|destroy`, `aws s3 cp|rm`, and `fs cp`.

### 9a. `CLAUDE.md` (repo root), outline

```
# databricks-learning
Standalone Databricks-on-AWS learning lab: multi-source payer ingestion → Unity Catalog →
Lakeflow bronze → dbt-databricks common model → Lakeflow Jobs. Synthetic data only.

## Hard rules
- Standalone: nothing references any existing infra/account resources; all infra from terraform/ stacks.
- Region us-west-2 only. Serverless only (no classic clusters, no continuous pipelines, no serving endpoints).
- Budget cap $400 (trial credits). Tripwires $150/$250/$325 — at $325 stop and tear down. Run /cost-check before
  any expensive step (full dbt build, backfill, pipeline full refresh).
- Public repo: never commit account IDs, ARNs, workspace URLs, tokens, *.tfvars, state, profiles.yml.
- The user writes all lab code. Claude reviews, explains, and runs read-only commands only; mutating
  commands are handed to the user to run in their own terminal.

## Layout            (short map of terraform/, generators/, config/sources/, libs/, bundles/, docs/)
## Naming
- source_id: source_a|source_b|source_c… ; feeds: eligibility|medical|pharmacy|…
- Landing: s3://<landing-bucket>/<source_id>/<feed>/yyyy/mm/dd/<source_id>_<feed>_<yyyymmdd>[_<seq>].<ext>
  → /Volumes/lab_prod/landing/<source_id>/…
- Catalogs lab_dev / lab_preprod / lab_prod (dev → main → release); schemas landing, <source_id>_bronze (one per source), silver (sources unioned), gold (marts), observation (pipeline health), outbound (return files)
- Bronze tables: <source_id>_bronze.<feed> (+ <feed>_quarantine); silver: stg_<source_id>__<feed>, int_*; gold: marts
## Commands         (CLI profiles acct/lab, bundle targets dev/prod, dbt targets, make targets)
## Skills           (index of §9b with one line each)
```

### 9b. Skills (`.claude/skills/<name>/SKILL.md`), checklist runbooks

| Skill | Purpose |
|---|---|
| **`/add-payer-files`** | The source-onboarding runbook. **Mode A: new drop for an existing source.** Mode B: **onboard a new source**. Claude walks the checklist one step at a time, verifying each box before moving on (details below). |
| `/phase-check <n>` | Checks the "Done when" criteria for a plan phase with read-only commands. Records pass/fail and notes in `docs/progress.md`, then names the next step. |
| `/cost-check` | Queries `system.billing.usage`, totals credits used against $400, and shows the trend by SKU. Warns at $250 and says **STOP** at $325. Other skills call it first. |
| `/tf-stack <stack>` | Terraform runbook: fmt → validate → plan. Claude reads the plan and flags anything outside us-west-2, untagged resources, non-serverless compute, destroys, or references to non-lab resources. Then it hands you the `apply` command and confirms there's no drift afterward. |
| `/bundle-ship <bundle> <dev\|preprod\|prod>` | For dev, validate → you deploy → Claude verifies. For preprod and prod, it walks the merge to `main` and the `main` → `release` PR, then verifies the CI deploy, the run state and the output tables. |
| `/dbt-dev-loop <selector>` | Build the selected models in `lab_dev` (with dbt MCP), run tests, compare row counts with `lab_prod`, and check contracts. Uses a scoped `--select` to save cost. |
| `/break-it <drill>` | Runs one Phase 4b drill: shows the fixture and its expected outcome, hands you the drop command, then verifies that exactly that outcome happened (pipeline state, event-log expectation counts, quarantine rows, job/task status, alert fired) and logs pass/fail. |
| `/triage-run <job or pipeline>` | Failure runbook: get-run or event log → classify the failure (schema drift / rescued data / expectation / dbt test / permissions) → fix checklist → hands you `repair-run`. |
| `/dq-scorecard` | Compares dbt test failures with the Tuva `issue_manifest.csv` and reports the catch rate per defect class. |
| `/teardown` | Ordered destroy (40 → 10), then checks that the AWS tag query is empty, that no Databricks resources remain, and that the subscription isn't converting to paid. |

**`/add-payer-files`, step by step:**

Mode A, a new drop for an existing source:
1. `/cost-check`.
2. Confirm `source_id`, feed and drop date.
3. Validate file names against the naming convention.
4. Run a local sanity check (row count, header, delimiter or encoding).
5. You run `aws s3 cp …`. Claude checks it landed (`aws s3 ls`, `databricks fs ls dbfs:/Volumes/...`).
6. Confirm the file-arrival trigger fired, or you run the job.
7. Check bronze row count parity against `read_files`.
8. Check `_rescued_data` is null and no expectation failed.
9. Run the downstream dbt build/tests for `tag:<source_id>+`.
10. Check freshness in `observation.feed_freshness`.
11. Log the result in `docs/progress.md`.

Mode B, onboarding a new source. Steps 1–3 are the same as Mode A, then:
4. Profile the sample with `read_files` in the SQL Editor, and record the quirks.
5. You write `config/sources/<id>.yml` (format, delimiter, path glob, cadence, PK, PHI columns). Claude reviews it against the profile.
6. Terraform: add a `source_landing` module call (prefix, external volume, grants), then `/tf-stack 30`.
7. Land the sample (Mode A steps 5–8).
8. You write the dbt `sources.yml`, the `stg_<id>__*` models and tests. Claude reviews the column mapping to the common contract.
9. Add the source to `union_sources()`.
10. Contracts and tests pass in dev (`/dbt-dev-loop`).
11. Extend the row filter to the new `source_id` and set PHI column tags.
12. Set the freshness SLA for the new cadence.
13. Update `docs/onboarding-a-new-source.md` with anything new.
14. Ship through `main` (preprod, golden drop green) → `release` (prod) using `/bundle-ship`.
15. `/phase-check`.

### 9c. MCP servers (`.mcp.json`, no secrets; credentials come from env or CLI profiles)

| Server | What it gives Claude | Notes |
|---|---|---|
| **dbt MCP** (`dbt-mcp` from dbt Labs, run with `uvx dbt-mcp`) | Run `dbt list/compile/show/build/test` against `bundles/transform/dbt`, read the manifest and lineage, show model SQL and results | Configure for dbt Core: `DBT_PROJECT_DIR`, `DBT_PATH`, profile `lab` / target `dev`. Turn off dbt Cloud/Semantic Layer tools. Mutating dbt commands only target `dev`. |
| **Databricks managed MCP** (workspace-hosted: Databricks SQL, Genie space, UC functions) | Run read-only SQL against `lab_dev`/`lab_prod`, ask the Genie space, list catalog objects | OAuth through the CLI profile. Copy the exact server URLs from the workspace's MCP/Agents page in Phase 3. **Every query bills the warehouse**, so it counts toward the budget. |
| **Terraform MCP** (`hashicorp/terraform-mcp-server`) | Look up current `databricks/databricks` and `hashicorp/aws` provider docs and resource arguments | Docs only, no state access. Catches stale-docs mistakes like the bundle rename. |
| *Optional:* AWS Documentation MCP (`awslabs`) | IAM and S3 policy reference for the UC storage credential trust policy | Docs only. |

### 9d. `.claude/settings.json` (enforces the working agreement)

- **allow:**
  - Databricks: `databricks * list*`, `databricks * get*`, `databricks bundle validate*`, `databricks bundle summary*`, `databricks fs ls*`, `databricks current-user*`
  - Terraform: `terraform fmt*`, `terraform validate*`, `terraform plan*`
  - AWS: `aws s3 ls*`, `aws sts get-caller-identity*`, `aws resourcegroupstaggingapi get-resources*`
  - `uv run pytest*`
  - The dbt MCP read tools
- **deny:**
  - Terraform: `terraform apply*`, `terraform destroy*`
  - Databricks: `databricks bundle deploy*`, `databricks bundle run*`, `databricks bundle destroy*`, `databricks fs cp*`, `databricks fs rm*`
  - AWS: `aws s3 cp*`, `aws s3 rm*`, `aws s3 sync*`
  - Git: `git push*`
- **Hook (optional):** a `PreToolUse` guard that blocks writing files that look like secrets (`*.tfvars`, `.databrickscfg`, `profiles.yml` with tokens) in the repo.

## Verification

- **Infra:** after apply, `terraform plan` shows no changes for every stack, and destroy followed by re-apply succeeds. AWS resources exist only in us-west-2 and carry the lab tags (`aws resourcegroupstaggingapi get-resources --tag-filters Key=project,Values=databricks-learning --region us-west-2`). After destroy, that query returns nothing.
- **Ingestion:** bronze row counts match source file row counts (checked with SQL against `read_files`).
- **dbt:** `dbt build` is green for all 3 sources, contracts are enforced, unit tests pass, and the DQ catch rate is reported.
- **Governance:** restricted principals get filtered or masked results.
- **Orchestration:** a bare `aws s3 cp` of a new file drives the whole chain through to dashboard freshness.
- **CI/CD:** the MR pipeline is green and merging deploys to prod.

## Getting started

1. Install the CLIs: `brew tap databricks/tap && brew install databricks`, `terraform`, `uv`.
2. Log in with an AWS profile that has IAM and S3 rights. Paste the output of `databricks --version`,
   `terraform version` and `aws sts get-caller-identity` into the Claude session.
3. Write the `.gitignore` additions and `terraform/00-bootstrap` (state bucket, budget alarm, GitHub OIDC role).
   Then run `/tf-stack 00` for review.
