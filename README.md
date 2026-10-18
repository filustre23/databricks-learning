# databricks-learning

**A hands-on Databricks lakehouse lab: multiple payers send differently shaped eligibility and claims feeds,
and they are landed, governed, tested and unified into one common model.**

Built to learn Databricks end to end (infrastructure, ingestion, storage, SQL, dbt, orchestration, governance
and CI/CD) using the CLI, the UI and code. All data is synthetic, generated from public healthcare data generators.

## Architecture

```
 PAYERS (simulated)          AWS us-west-2                      DATABRICKS · Unity Catalog
 ──────────────────          ─────────────                      ─────────────────────────────────────────────
 source_a  Medicare FFS  ┐
 source_b  Commercial    ├─▶  s3://<landing>/<source>/…  ──▶  landing        one volume per source
 source_c  Regional plan ┘                                         │  file-arrival trigger
                                                                   ▼
                                                      <source>_bronze  one Lakeflow pipeline per source
                                                                   │   Auto Loader · expectations · quarantine
                                                                   ▼
                                                      silver           dbt: per-source staging → UNION → conformed
                                                                   ▼
                                                      gold             eligibility · member_months ·
                                                                       medical_claim · pharmacy_claim
                                                      observation      pipeline health · data quality
                                                      outbound         reconciliation · return files → payers
```

| Concern | How it's done |
|---|---|
| Infrastructure | Terraform: `databricks` (account and workspace) + `aws` providers; serverless workspace |
| Ingestion | Volumes + Auto Loader in Lakeflow pipelines, driven by a per-source YAML registry |
| Transformation | dbt-databricks on a serverless SQL warehouse |
| Orchestration | Lakeflow Jobs: file arrival → pipeline → dbt → SQL checks → alerts |
| Governance | Unity Catalog grants, row filters, column masks, tags, lineage |
| Environments | `lab_dev` → `lab_preprod` (branch `main`) → `lab_prod` (branch `release`) |
| Deploys | Asset Bundles via GitHub Actions with OIDC (no stored secrets) |
| Python | uv workspace (`uv sync`, `uv run`, `uv build`) |

## Data sources

| Source | Based on | Format | Cadence |
|---|---|---|---|
| `source_a` | [CMS Synthetic Medicare FFS claims](https://data.cms.gov/collection/synthetic-medicare-enrollment-fee-for-service-claims-and-prescription-drug-event) | Pipe-delimited RIF | Monthly |
| `source_b` | [Tuva synthetic data generator](https://github.com/tuva-health/synthetic-data-generator), Aetna layout | CSV.gz + fixed-width | Weekly |
| `source_c` | Tuva synthetic data generator, HCCI layout | Parquet | Daily |

The unified model follows the shape of the [Tuva core data model](https://thetuvaproject.com/core-data-model/overview).

## Repository layout

| Path | Contents |
|---|---|
| `terraform/` | Stacks `00-bootstrap` → `40-workspace-config`, plus modules |
| `generators/` | Scripts that produce and replay each source's file drops |
| `config/sources/` | Feed registry: one YAML per source (format, schema, cadence, expectations) |
| `libs/source_ingest/` | Shared Python package (parsers, registry), shipped as a wheel |
| `bundles/` | Asset Bundles: `ingestion`, `transform` (dbt), `ops` (dashboards, alerts) |
| `tests/` | Unit tests, golden-drop integration tests, break-it drills |
| `docs/` | [Plan](docs/plan.md), [platform equivalents (GCP · Snowflake · Databricks)](docs/platform-equivalents.md), ADRs |
| `.claude/` | Claude Code skills (runbooks) and settings |

## Working with Claude Code

The repo is set up for learning with Claude Code as a coach. [`CLAUDE.md`](CLAUDE.md) defines the working agreement:
you write the code and run anything that changes state; Claude reviews, explains and verifies with read-only commands.

| Skill | Purpose |
|---|---|
| `/add-payer-files` | Runbook: land a new drop, or onboard a new source end to end |
| `/break-it` | Run a breakage drill and verify the expected alert or quarantine happened |
| `/tf-stack` · `/bundle-ship` | Review and ship Terraform stacks and bundles through dev → preprod → prod |
| `/dbt-dev-loop` · `/triage-run` · `/dq-scorecard` | Develop, debug and score data quality |
| `/cost-check` · `/phase-check` · `/teardown` | Budget, progress and clean shutdown |

## Budget

The lab is designed to run inside a Databricks trial on about $400 of credits: serverless only, small data,
triggered pipelines, and budget tripwires at $150 / $250 / $325. See [docs/plan.md §8](docs/plan.md).

## Related

[**knowledge-builder**](https://github.com/filustre23/knowledge-builder) is a set of interactive learning modules
(Databricks, claims data analytics) that run against this environment.
