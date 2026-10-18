# GCP → Databricks (on AWS) equivalents

A translation guide for someone fluent in a GCP data stack (BigQuery, GCS, Dataform/dbt, Composer or Airflow,
Cloud Run, Terraform) who is learning Databricks on AWS. "≈" means close but not identical; the *Differences*
column is where the surprises are.

This lab runs Databricks **on AWS**, so the storage, IAM and networking rows map to AWS services rather than
GCP ones.

---

## 1. Mental model shifts

| In GCP / BigQuery you're used to… | In Databricks… |
|---|---|
| Storage and compute are both "just BigQuery". You never pick compute. | You **choose compute** for every workload: a SQL warehouse, serverless jobs, a pipeline, a cluster. Data lives in open files (Delta/Parquet) in object storage. |
| Being able to read a table means being able to query it (Data Viewer + Job User). | **Data permissions and compute permissions are separate.** You need `SELECT` on the table *and* `CAN USE` on a warehouse or compute. |
| Files in GCS and tables in BigQuery are governed separately. | **Files are governed alongside tables.** Volumes live in Unity Catalog next to tables, with the same grants and lineage. |
| The project is both the billing unit and the data container. | The **account** handles billing and identity, the **workspace** is where you work, the **metastore and catalogs** contain data. They're separate concepts. |
| Storage maintenance is invisible. | Delta tables need `OPTIMIZE` and `VACUUM`. **Predictive optimization** automates both for managed tables, but you should know they exist. |
| Streaming means Dataflow plus Pub/Sub, a separate product. | Streaming is part of the same engine: Auto Loader, streaming tables, Structured Streaming. |
| There's one console per project. | Two consoles: the **account console** (workspaces, metastore, identity, billing) and the **workspace UI** (data, compute, jobs, SQL). |

---

## 2. Resource hierarchy and namespace

| GCP | Databricks | Differences |
|---|---|---|
| Organization | Databricks **account** | The account owns workspaces, the metastore, users, groups, service principals and billing. |
| Folder | (none) | Group workspaces or catalogs by naming, tags or workspace bindings. |
| Project, as a billing and IAM boundary | **Workspace**, for compute and the UI | A workspace is where notebooks, jobs and warehouses live. Data isn't "in" a workspace. |
| Project, as a data container (`project.dataset.table`) | **Catalog** (`catalog.schema.table`) | Catalogs commonly map to environments (`lab_dev`, `lab_preprod`, `lab_prod`) or domains. |
| Region-level BigQuery storage | **Metastore**, one per region per account | The top of Unity Catalog. It's attached to workspaces in that region. |
| Dataset | **Schema** (also called a database) | Same role: groups tables and views and holds the default grants. |
| Table | Table: **managed** or **external** | Managed tables are stored in UC-managed storage and are dropped with the table. External tables point at your own S3 path. |
| View | View | UC views execute with the **owner's** rights, the way authorized views do. |
| Materialized view | Materialized view | Both refresh incrementally where they can. Databricks MVs run on serverless pipelines. |
| (Dataflow streaming output) | **Streaming table** | A table fed incrementally by a stream, such as Auto Loader on a volume. |
| Routine / UDF | **Function** (SQL or Python UDF in UC) | Governed and shareable. It's also the hook for row filters and column masks. |
| GCS bucket or folder of raw files | **Volume** (managed or external) | Path `/Volumes/<catalog>/<schema>/<volume>/…`. Grants work on it like on a table. |
| BigQuery ML model | **Registered model** in UC (MLflow) | Lives in the catalog next to the data. |
| `INFORMATION_SCHEMA` per dataset or region | `<catalog>.information_schema.*`, `system.information_schema.*` | Standard information schema per catalog, plus a cross-catalog view. |

---

## 3. Storage and table format

| GCP / BigQuery | Databricks on AWS | Differences |
|---|---|---|
| GCS | **S3** | Exposed through UC external locations and volumes. |
| BigQuery native storage (proprietary, managed) | **Delta Lake** (open: Parquet files plus a transaction log) | Any engine that reads Delta can read it. **UniForm** or managed Iceberg also exposes tables as Iceberg. |
| BigLake / external tables over GCS | **External tables** and **volumes** over S3 | `read_files()` queries raw files directly, with no table needed. |
| Partitioning (`PARTITION BY date`) | **Liquid clustering** (`CLUSTER BY (col…)`, or `CLUSTER BY AUTO`) | Liquid clustering is preferred over partitions. Partitioning still exists but is discouraged below roughly 1 TB. |
| Clustering (up to 4 columns) | Liquid clustering (up to 4 keys); Z-order on older tables | Keys can be changed later without rewriting the table. |
| Time travel: `FOR SYSTEM_TIME AS OF`, 2–7 day window | `VERSION AS OF n` / `TIMESTAMP AS OF ts` | Retention is governed by log retention and `VACUUM` (7 days of files by default). `DESCRIBE HISTORY` shows every commit. |
| Table snapshots and clones | `SHALLOW CLONE` / `DEEP CLONE` | Shallow clones are near-instant and good for testing against a copy of prod. |
| Partition and table expiration | (no auto-expire) | Schedule a `DELETE` followed by `VACUUM`, or drop old tables. |
| Active vs long-term storage pricing | S3 storage pricing (lifecycle rules for external data) | Compute and storage are billed separately: S3 bills storage, DBUs bill compute. |
| Wildcard tables (`events_*` + `_TABLE_SUFFIX`) | (none) | Use a single table with a clustering key instead of sharded tables. |
| Search indexes / vector search | **Vector Search** indexes | Synced from a Delta table. |

---

## 4. Compute

| GCP | Databricks | Differences |
|---|---|---|
| BigQuery on-demand (billed per byte scanned) | **Serverless SQL warehouse** (billed per DBU per second while it's running) | You pay for warehouse uptime, not bytes scanned. Auto-stop is your cost control. |
| BigQuery Editions reservations / slots | SQL warehouse **size** (2X-Small … 4X-Large) and **min/max clusters** | Size controls per-query power; cluster count controls concurrency. |
| Dataproc (Spark clusters) | Classic clusters, **serverless jobs compute** and serverless notebooks | Serverless needs no cluster config and starts in seconds. Classic clusters run in your own cloud account. |
| Dataproc Serverless batches | **Serverless job tasks** (Python script, wheel, notebook) | Runtime is set by a serverless *environment version*. |
| Cloud Run jobs (containerized batch) | **Lakeflow Job** tasks on serverless (Python wheel or script) | No container image to build. Code ships as files or wheels in an Asset Bundle. |
| Cloud Run services / App Engine | **Databricks Apps**, or **Model Serving** for models | Apps host small Python web apps (Streamlit, Dash, FastAPI) next to the data. |
| Cloud Functions | (no direct equivalent) | Use a file-arrival-triggered job, a Databricks App, or AWS Lambda calling the Databricks API. |
| Vertex AI Workbench | **Notebooks** (serverless or cluster) | Multi-language (SQL, Python, Scala, R) and Git-backed through Git folders. |
| Local development against BigQuery | **Databricks Connect**, the VS Code extension, the dbt adapter | Run PySpark locally against remote serverless compute. |

---

## 5. Ingestion and streaming

| GCP | Databricks | Differences |
|---|---|---|
| GCS + Cloud Functions / Eventarc on object finalize | **Auto Loader** (`cloudFiles`) and **file-arrival triggers** on jobs | Auto Loader tracks which files it has processed, infers and evolves the schema, and keeps unparseable data in `_rescued_data`. |
| `bq load` / load jobs | `COPY INTO`, `read_files()`, or an Auto Loader streaming table | `COPY INTO` is idempotent (it skips files it has already loaded). |
| BigQuery Data Transfer Service | **Lakeflow Connect** (managed SaaS connectors) | Managed connectors for things like Salesforce, Workday and Google Analytics. |
| Datastream (CDC from Postgres, MySQL, SQL Server, Oracle) | **Lakeflow Connect database connectors** (CDC); `AUTO CDC INTO` in pipelines | `AUTO CDC` applies change feeds, including SCD type 1 and 2, inside a pipeline. |
| Dataflow (Apache Beam) | **Lakeflow pipelines** (declarative) or Structured Streaming | Declarative: you write the tables and their quality rules; the engine handles orchestration, retries and checkpoints. |
| Pub/Sub | (no native queue) Kafka, Kinesis or MSK as a streaming source | Databricks reads from and writes to queues but doesn't host one. |
| Dataplex data quality scans | **Pipeline expectations** (`expect`, `expect_or_drop`, `expect_or_fail`), data quality monitoring | Expectations run inline with ingestion, and their metrics land in the pipeline event log. |
| BigQuery Omni / federated queries (Cloud SQL, Spanner) | **Lakehouse Federation**: foreign catalogs over Postgres, MySQL, SQL Server, Snowflake, BigQuery and others | Query external systems in place, governed in UC. |
| Analytics Hub (data exchange) | **Delta Sharing** and **Marketplace** | An open protocol: recipients don't need Databricks. |
| Upload a CSV in the console | **Data Ingestion → Create or modify table** (UI upload) | Fine for seeds and reference lists. |

---

## 6. Transformation (SQL, dbt, Dataform)

| GCP | Databricks | Differences |
|---|---|---|
| Dataform (SQLX) | **dbt-databricks**, or Lakeflow pipelines in SQL | dbt runs as a native task type in Lakeflow Jobs, on a SQL warehouse. |
| `dbt-bigquery` | `dbt-databricks` | See the dbt config table below. |
| Scheduled queries | A SQL task in a job, or a scheduled query in the SQL editor | Jobs add dependencies, retries and alerts. |
| BigQuery scripting (`DECLARE`, `IF`, `LOOP`) | **SQL scripting** (`BEGIN … END`, `DECLARE`, `IF`, `WHILE`) | Recent runtimes and warehouses support it. |
| Stored procedures | SQL scripting / procedures, or a Python task | Prefer pipeline or job tasks for anything non-trivial. |
| Dataform assertions | dbt tests, model contracts and unit tests, plus pipeline expectations | Use expectations at ingestion and dbt tests in modeling. |

### dbt config: `dbt-bigquery` → `dbt-databricks`

| dbt-bigquery | dbt-databricks | Notes |
|---|---|---|
| `project` / `database` | `catalog` (alias `database`) | Set per target in `profiles.yml`. |
| `dataset` / `schema` | `schema` | Same meaning. |
| `method: oauth` / `service-account` keyfile | `host`, `http_path` (the warehouse), OAuth (`auth_type: oauth`) or a token | `http_path` picks the compute. |
| `partition_by={'field': 'd', 'data_type': 'date'}` | `liquid_clustered_by=['d']`, or `partition_by=['d']` (discouraged when small) | Prefer liquid clustering. |
| `cluster_by=['a','b']` | `liquid_clustered_by=['a','b']` (or `zorder` on older tables) | |
| `incremental_strategy`: `merge`, `insert_overwrite` (by partition), `microbatch` | `merge`, `append`, `insert_overwrite`, **`replace_where`**, `microbatch` | `replace_where` replaces exactly the rows that match a predicate, such as one month of data. |
| `require_partition_filter`, `partition_expiration_days` | (none) | Handle these with clustering and retention jobs. |
| `labels` | `tblproperties`, UC tags (`databricks_tags`) | Tags are searchable and drive governance. |
| `grant_access_to` (authorized views) | `grants` config, plus UC view ownership | UC views run with the owner's rights. |
| `materialized='materialized_view'` | `materialized_view`, and also **`streaming_table`** | Streaming tables refresh incrementally from append-only sources. |
| Python models via Dataproc | Python models run natively (serverless or a cluster) | No separate Spark service. |
| `persist_docs` → table and column descriptions | `persist_docs` → UC comments | Shows up in Catalog Explorer and Genie. |
| `query_comment` / job labels for cost attribution | Query tags, serverless budget policies, `system.query.history` | Attribute warehouse cost by tag or query. |

---

## 7. SQL dialect: GoogleSQL → Databricks SQL

| GoogleSQL (BigQuery) | Databricks SQL | Watch out |
|---|---|---|
| `` `project.dataset.table` `` | `catalog.schema.table` (backticks only for special characters) | |
| `INT64`, `FLOAT64`, `NUMERIC`, `BIGNUMERIC`, `STRING`, `BYTES`, `BOOL` | `BIGINT`, `DOUBLE`, `DECIMAL(38,9)`, `DECIMAL(38,x)`, `STRING`, `BINARY`, `BOOLEAN` | |
| `JSON` type, `JSON_VALUE(j, '$.a')` | `VARIANT` (`parse_json`), `j:a::string`, `get_json_object(j, '$.a')` | The colon path syntax is the idiomatic form. |
| `SAFE_CAST(x AS INT64)` | `try_cast(x AS BIGINT)` | |
| `SAFE_DIVIDE(a, b)` | `try_divide(a, b)` | |
| `IFNULL(a, b)` | `ifnull(a, b)` / `coalesce` / `nvl` | |
| `DATE_DIFF(end, start, DAY)` | `datediff(end, start)`, or `timestampdiff(DAY, start, end)` | Argument order differs between the two Databricks forms. |
| `DATE_ADD(d, INTERVAL 1 DAY)` | `date_add(d, 1)`, or `d + INTERVAL 1 DAY` | |
| `DATE_TRUNC(d, MONTH)` | `date_trunc('MONTH', d)` | **The arguments are flipped.** |
| `TIMESTAMP_DIFF(a, b, HOUR)` | `timestampdiff(HOUR, b, a)` | The unit comes first, then start and end. |
| `FORMAT_DATE('%Y-%m', d)` | `date_format(d, 'yyyy-MM')` | Java datetime patterns, not strftime. |
| `PARSE_DATE('%Y%m%d', s)` | `to_date(s, 'yyyyMMdd')` | Java patterns again. |
| `EXTRACT(YEAR FROM d)` | `EXTRACT(YEAR FROM d)` or `year(d)` | |
| `GENERATE_DATE_ARRAY(a, b)` | `sequence(a, b, INTERVAL 1 DAY)` | |
| `UNNEST(arr)` | `explode(arr)` (in SELECT, `LATERAL VIEW`, or as a table function) | `posexplode` gives the index too. |
| `ARRAY_AGG(x)` | `array_agg(x)` / `collect_list(x)` | `collect_set` gives distinct values. |
| `STRING_AGG(x, ',')` | `string_agg` / `listagg` on newer runtimes, or `array_join(collect_list(x), ',')` | |
| `REGEXP_CONTAINS(s, r)` | `regexp_like(s, r)` / `s RLIKE r` | |
| `REGEXP_EXTRACT(s, r)` | `regexp_extract(s, r, 1)` | Takes a group index. |
| `SELECT * EXCEPT (a)` | `SELECT * EXCEPT (a)` | Supported. |
| `QUALIFY` | `QUALIFY` | Supported. |
| `MERGE … WHEN NOT MATCHED BY SOURCE` | `MERGE INTO … WHEN NOT MATCHED BY SOURCE` | Supported. |
| `CREATE TEMP TABLE` in scripts | `CREATE TEMPORARY VIEW` (or temp tables where available) | |
| `FARM_FINGERPRINT(s)` | `xxhash64(s)` / `hash(s)` / `sha2(s, 256)` | Different hash values, so don't compare across systems. |
| `CURRENT_DATETIME()` | `current_timestamp()` (session time zone) | Mind the session time zone. |
| `ML.PREDICT`, `ML.GENERATE_TEXT` | `ai_query()`, `ai_classify`, `ai_extract`, `ai_summarize` | AI functions call serving endpoints and bill per token. |

---

## 8. Orchestration and deployment

| GCP | Databricks | Differences |
|---|---|---|
| Cloud Composer / Airflow DAG | **Lakeflow Job** (a DAG of tasks) | Task types: notebook, Python script or wheel, SQL, **dbt**, pipeline, run-job, condition (if/else), for-each. |
| Cloud Scheduler | Job **schedule** (cron) | Defined in the bundle YAML. |
| GCS / Eventarc trigger | **File-arrival trigger**, **table-update trigger** | Native, with no extra service. |
| Airflow sensors | Table-update triggers, condition tasks, run-job task dependencies | |
| Airflow "clear task" and re-run | **Repair run** | Re-runs only the failed tasks and their dependents. |
| Airflow XCom | **Task values** (`dbutils.jobs.taskValues`), job parameters | |
| Cloud Build / container images plus Terraform for deploys | **Asset Bundles** (`databricks.yml`, `bundle deploy -t <target>`) | Bundles deploy jobs, pipelines, dashboards and code together, with per-target overrides. |
| Per-environment GCP projects | Bundle **targets** (dev / preprod / prod) and a catalog per environment | `mode: development` prefixes resources with your username and pauses schedules. |
| Airflow pools / concurrency | `max_concurrent_runs`, queueing, warehouse max clusters | |

---

## 9. Governance and security

| GCP | Databricks | Differences |
|---|---|---|
| Cloud IAM roles on project or dataset | **UC privileges**: `USE CATALOG`, `USE SCHEMA`, `SELECT`, `MODIFY`, `CREATE TABLE`, `READ VOLUME`, `WRITE VOLUME`, … | Grants inherit downward: catalog → schema → object. Use SQL `GRANT` or Terraform `databricks_grants`. |
| `roles/bigquery.dataViewer` | `USE CATALOG` + `USE SCHEMA` + `SELECT` | |
| `roles/bigquery.dataEditor` | + `MODIFY`, `CREATE TABLE` | |
| `roles/bigquery.jobUser` | `CAN USE` on a SQL warehouse, or a compute entitlement | **Compute access is a separate grant.** |
| Dataset or table owner | Object **owner** (a user, group or service principal) | Make groups or service principals owners, not individuals. |
| Data Catalog / Dataplex (metadata, search, lineage) | **Unity Catalog / Catalog Explorer** | Lineage is captured automatically down to column level, including notebooks and jobs. |
| Policy tags (column-level security) | **Column masks** (a UC function attached to the column) | |
| Dynamic data masking | Column masks; dynamic views with `is_account_group_member()` | |
| Row-level access policies | **Row filters** (a UC function attached to the table) | |
| Authorized views and datasets | Views (owner's rights) and grants on views only | |
| Resource labels / tags | **Tags** on catalogs, schemas, tables and columns | Searchable, and you can write governance rules on them. |
| Cloud Audit Logs (data access) | **`system.access.audit`** | Queryable with SQL. |
| VPC Service Controls | Serverless **network policies**, PrivateLink, IP access lists, workspace bindings | |
| Private Service Connect | **AWS PrivateLink** (front-end and back-end) | |
| CMEK (Cloud KMS) | Customer-managed keys (AWS KMS) for managed services and storage | |
| Assured Workloads / HIPAA config | **Compliance security profile** (HIPAA and others) plus the enhanced security add-on | Needed before PHI goes into a workspace. |

---

## 10. Identity and secrets

| GCP | Databricks / AWS | Differences |
|---|---|---|
| Google Workspace users and groups | Account **users** and **groups** (synced by SCIM from your IdP) | Manage groups at the account level, not per workspace. |
| Service account | **Service principal** (Databricks), **IAM role** (AWS) | Service principals run jobs (`run_as`) and CI deploys. |
| Service account key (JSON) | OAuth M2M client secret, or a PAT (avoid) | Prefer OAuth and federation over long-lived secrets. |
| Workload Identity Federation (GitHub → GCP) | **Workload identity federation** (GitHub OIDC → service principal) and GitHub OIDC → AWS IAM role | Federation policies scope a service principal to a branch or environment. |
| Service account impersonation | `run_as` on jobs and pipelines; IAM `AssumeRole` on AWS | |
| A service account's access to GCS | **Storage credential** (an IAM role) + **external location** | UC brokers S3 access, so users never get raw S3 credentials. |
| Secret Manager | **Secret scopes** (Databricks-backed), or AWS Secrets Manager | Read with `dbutils.secrets.get(scope, key)`. Values are redacted in output. |
| `gcloud auth application-default login` | `databricks auth login --host …` (OAuth U2M), saved to `~/.databrickscfg` profiles | |

---

## 11. Observability and cost

| GCP | Databricks | Differences |
|---|---|---|
| `INFORMATION_SCHEMA.JOBS` (query history, bytes, slots) | **`system.query.history`** and the Query History UI with query profiles | Query profiles show the operator tree, spill and skew. |
| Billing export to BigQuery | **`system.billing.usage`** + `system.billing.list_prices` | DBUs by SKU, job, warehouse or pipeline, plus custom tags. |
| Budgets and alerts | Account **budgets**, serverless **budget policies** | Budget policies tag serverless usage by team or project. |
| Cloud Monitoring alert policies | **SQL Alerts**, job and pipeline **notifications**, job **health rules** | Destinations: email, Slack, Teams, webhook, PagerDuty. |
| Cloud Logging for job logs | Job run output, **pipeline event log** (publishable to a UC table), cluster logs | The event log holds expectation metrics, lineage and errors. |
| Composer / Airflow UI for run state | **Jobs & Pipelines → Runs** (matrix view, repair) | |
| Dataplex auto data quality | **Data quality monitoring** (anomaly detection on freshness and completeness), expectations | Monitoring bills serverless compute. |

---

## 12. BI, AI and ML

| GCP | Databricks | Differences |
|---|---|---|
| Looker Studio / Looker | **AI/BI Dashboards** | Exportable as `.lvdash.json` and deployable with bundles. |
| Looker semantic layer (LookML) | **Metric views** in UC, the dbt semantic layer | Metric views are YAML-defined measures and dimensions in the catalog. |
| Gemini in BigQuery / conversational analytics | **Genie** (AI/BI Genie spaces, Genie agents) | Ask questions in natural language over the tables you curate, guided by instructions and example SQL. |
| Vertex AI Model Garden / endpoints | **Model Serving** and foundation model APIs, **AI Gateway** | AI Gateway handles rate limits, logging and guardrails for endpoints. |
| Vertex AI Experiments | **MLflow Experiments** | |
| Vertex AI Feature Store | **Feature tables** in UC, online tables | |
| Vertex AI Model Registry | **Models in UC** (MLflow) | |
| Vertex AI Agent Builder | **Agents** (Mosaic AI Agent Framework), Playground | |

---

## 13. CLI and infrastructure as code

| GCP CLI | Databricks / AWS CLI | Notes |
|---|---|---|
| `gcloud auth login` | `databricks auth login --host <url> --profile <name>` | Profiles are kept in `~/.databrickscfg`. |
| `gcloud config configurations activate` | `-p <profile>`, or `DATABRICKS_CONFIG_PROFILE` | |
| `bq ls project:` | `databricks catalogs list` / `schemas list <catalog>` / `tables list <catalog> <schema>` | |
| `bq show project:dataset.table` | `databricks tables get <catalog>.<schema>.<table>` | |
| `bq mk --dataset` | `databricks schemas create <name> <catalog>` | Better done in Terraform. |
| `bq query 'SELECT …'` | SQL Editor, `databricks api post /api/2.0/sql/statements` (statement execution), the Databricks SQL MCP | No first-class `query` subcommand; use the API or the editor. |
| `gcloud storage ls/cp gs://…` | `databricks fs ls/cp dbfs:/Volumes/…`, `aws s3 ls/cp s3://…` | |
| `gcloud run jobs execute` | `databricks bundle run <job>` / `databricks jobs run-now <id>` | |
| `gcloud builds submit` + deploy | `databricks bundle validate` → `bundle deploy -t <target>` | |
| `gcloud projects get-iam-policy` | `databricks grants get <type> <name>` | |
| Terraform `google` provider | Terraform **`databricks`** provider (account and workspace level) + **`aws`** provider | Account level covers workspaces, the metastore, groups and service principals. Workspace level covers catalogs, schemas, volumes, grants and warehouses. |
| `google_bigquery_dataset` | `databricks_schema` | |
| `google_bigquery_table` | `databricks_sql_table` (prefer dbt or pipelines for tables) | |
| `google_storage_bucket` | `aws_s3_bucket` + `databricks_external_location` / `databricks_volume` | |
| `google_service_account` | `databricks_service_principal` / `aws_iam_role` | |
| `google_*_iam_member` | `databricks_grants` / `databricks_permissions` | Grants cover data; permissions cover jobs, warehouses and other workspace objects. |
| `google_cloud_scheduler_job` | `resources.jobs.<name>.schedule` in a bundle | |

---

## 14. Where this lab uses each piece

| Lab layer | Databricks feature | GCP mental equivalent |
|---|---|---|
| Payer drops | S3 + UC external volume | GCS bucket with governed access |
| Bronze (`<source_id>_bronze`, one schema per source) | One Lakeflow pipeline per source + Auto Loader + expectations + quarantine | A Dataflow job per source writing to its own raw dataset + GCS trigger + Dataplex DQ |
| Silver (sources unioned) and gold (marts) | dbt-databricks on a serverless SQL warehouse | dbt-bigquery / Dataform on BigQuery, with staging and marts datasets |
| Environments | `lab_dev` / `lab_preprod` / `lab_prod` catalogs + bundle targets | A project or dataset per environment |
| Orchestration | Lakeflow Jobs (file-arrival trigger → pipeline → dbt → SQL check) | Composer DAG + Cloud Scheduler + sensors |
| Governance | UC grants, row filters, column masks, tags, lineage | IAM + row access policies + policy tags + Dataplex |
| Alerts | Pipeline and job notifications, SQL Alerts | Cloud Monitoring alert policies |
| Cost | `system.billing.usage`, budgets, budget policies | Billing export + budgets |
| Infrastructure | Terraform `databricks` + `aws` providers | Terraform `google` provider |
| Deploys | Asset Bundles via GitHub Actions (OIDC) | Cloud Build / GitHub Actions with Workload Identity Federation |
