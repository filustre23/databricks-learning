# Platform equivalents: GCP · Snowflake · Databricks

The same concept across Google Cloud (BigQuery and the data services around it), Snowflake, and Databricks.
This lab runs Databricks **on AWS**, so the Databricks column names AWS services for storage, IAM and networking.
The *Watch out* notes under each table cover the differences that cause bugs.

---

## 1. Mental model

| GCP | Snowflake | Databricks |
|---|---|---|
| Compute is invisible: BigQuery allocates slots per query | You pick a **virtual warehouse** (size and cluster count) | You pick compute per workload: **SQL warehouse**, serverless jobs, pipelines, clusters |
| Proprietary managed storage | Proprietary managed storage (micro-partitions), plus Iceberg tables | **Open storage**: Delta Lake (Parquet plus a transaction log) in your object store |
| Data access and query rights are separate IAM roles (Data Viewer + Job User) | Data grants and `USAGE` on a warehouse are separate | Data grants (`SELECT`) and `CAN USE` on compute are separate |
| Files (GCS) are governed apart from tables | Files live in **stages**, governed as schema objects | Files live in **volumes**, governed in Unity Catalog next to tables |
| Storage maintenance is automatic | Storage maintenance is automatic | `OPTIMIZE`/`VACUUM` exist; **predictive optimization** automates them for managed tables |
| Streaming is a separate product (Dataflow + Pub/Sub) | Snowpipe Streaming, Streams + Tasks, Dynamic Tables | Same engine: Auto Loader, streaming tables, Structured Streaming |
| One console per project | One console (Snowsight) per account | Two consoles: **account console** (identity, workspaces, billing) and **workspace** (data, compute, jobs) |

---

## 2. Resource hierarchy and namespace

| GCP | Snowflake | Databricks |
|---|---|---|
| Organization | Organization | Account |
| Folder | (none) | (none) |
| Project (billing and IAM boundary) | Account | Workspace (compute and UI) plus account (billing and identity) |
| Region-level BigQuery storage | Account (region-bound) | **Metastore** (one per region) |
| Project as a data container: `project.dataset.table` | Database: `database.schema.table` | **Catalog**: `catalog.schema.table` |
| Dataset | Schema | Schema |
| Table | Permanent table | Managed table |
| External table / BigLake table | External table / Iceberg table | External table (or UniForm/Iceberg) |
| (none) | Transient table (no fail-safe) | Managed table with short retention |
| Temp table (script scope) | Temporary table | Temporary view |
| View | View | View |
| Authorized view | Secure view | View (runs with owner's rights) |
| Materialized view | Materialized view / **Dynamic table** | Materialized view / **streaming table** |
| UDF / routine | UDF / stored procedure | UC function (SQL or Python) / SQL scripting |
| GCS bucket or folder | Stage (internal or external) | **Volume** (managed or external) |
| BigQuery ML model | Model Registry object | Registered model in UC (MLflow) |
| `INFORMATION_SCHEMA` per dataset or region | `INFORMATION_SCHEMA` per database, plus the `SNOWFLAKE.ACCOUNT_USAGE` share | `<catalog>.information_schema`, `system.information_schema`, `system.*` tables |

**Watch out**
- A Databricks **catalog** plays the role of a BigQuery project or a Snowflake database: it's the top level of the data namespace.
- A Databricks **workspace** holds no data. It's where notebooks, jobs and warehouses live.

---

## 3. Storage and table features

| GCP | Snowflake | Databricks |
|---|---|---|
| GCS | Internal storage / external stage on S3, GCS or ADLS | S3 (in this lab), exposed through external locations and volumes |
| Capacitor (columnar, managed) | Micro-partitions (FDN, managed) | Delta Lake (Parquet + `_delta_log`) |
| Partitioning (`PARTITION BY`) | Automatic micro-partitioning | Liquid clustering (partitioning is possible but discouraged when tables are small) |
| Clustering (up to 4 columns) | Clustering keys + automatic clustering | **Liquid clustering** (`CLUSTER BY (…)` or `CLUSTER BY AUTO`) |
| Search indexes | Search Optimization Service | Data skipping, bloom filters, predictive I/O |
| Time travel (`FOR SYSTEM_TIME AS OF`, up to 7 days) | Time Travel (`AT`/`BEFORE`, 1–90 days) + 7-day Fail-safe | Time travel (`VERSION AS OF` / `TIMESTAMP AS OF`), bounded by `VACUUM` retention |
| Table snapshot / clone | Zero-copy `CLONE` | `SHALLOW CLONE` / `DEEP CLONE` |
| Undelete (time-travel restore) | `UNDROP TABLE` | `UNDROP TABLE` (managed tables, about 7 days) |
| Table or partition expiration | Data retention parameters | Scheduled `DELETE` + `VACUUM` |
| Wildcard / sharded tables (`events_*`) | (none) | (none); use one clustered table |
| Vector search | Cortex Search | Vector Search |

---

## 4. Compute

| GCP | Snowflake | Databricks |
|---|---|---|
| BigQuery on-demand (billed per TB scanned) | Virtual warehouse (credits per second, 60-second minimum) | Serverless SQL warehouse (DBUs per second while running) |
| Editions / slot reservations | Warehouse size XS–6XL | Warehouse size 2X-Small–4X-Large |
| Autoscaling slots | Multi-cluster warehouse (min/max clusters) | Min/max clusters on a SQL warehouse |
| (always on) | Auto-suspend / auto-resume | Auto-stop / auto-start |
| Dataproc (Spark) | Snowpark-optimized warehouse | Classic clusters / serverless jobs compute |
| Dataproc Serverless | Serverless tasks | Serverless job tasks |
| Cloud Run jobs | Snowpark Container Services jobs | Lakeflow Job tasks (Python script or wheel) |
| Cloud Run services | Snowpark Container Services, Streamlit in Snowflake | Databricks Apps, Model Serving |
| Cloud Functions | External functions / UDFs | File-arrival jobs, Apps, or a Lambda calling the API |
| Vertex AI Workbench / Colab Enterprise | Snowflake Notebooks | Notebooks |
| BigQuery DataFrames | Snowpark (Python, Java, Scala) | PySpark / Databricks Connect |

**Watch out**
- Snowflake and Databricks both bill for **compute uptime**: auto-suspend and auto-stop are your cost control.
- BigQuery on-demand bills for **bytes scanned**.

---

## 5. Ingestion and streaming

| GCP | Snowflake | Databricks |
|---|---|---|
| `bq load` / load jobs | `COPY INTO <table>` | `COPY INTO`, `read_files()` |
| GCS trigger → Cloud Function → load | **Snowpipe** (auto-ingest from stage events) | **Auto Loader** (`cloudFiles`) + file-arrival triggers |
| Pub/Sub → BigQuery subscription / Storage Write API | Snowpipe Streaming, Kafka connector | Structured Streaming from Kafka / Kinesis |
| Dataflow (Beam) | Streams + Tasks / Dynamic Tables | Lakeflow pipelines (declarative) / Structured Streaming |
| Datastream (CDC) | Openflow / database connectors (CDC) | Lakeflow Connect database connectors, `AUTO CDC INTO` |
| Data Transfer Service | Openflow / Marketplace connectors | Lakeflow Connect SaaS connectors |
| Dataplex data quality | Data metric functions (DMFs) | Pipeline **expectations**, data quality monitoring |
| Federated queries (Cloud SQL, Spanner), Omni | External tables, Iceberg catalog integration | **Lakehouse Federation** (foreign catalogs) |
| Analytics Hub | Secure Data Sharing, Marketplace, listings | Delta Sharing, Marketplace |
| Console CSV upload | Snowsight file load | Data Ingestion → Create or modify table |
| (none) | Schema evolution on `COPY` / Snowpipe | Schema evolution + `_rescued_data` in Auto Loader |

### Files: GCS ↔ stage ↔ volume

| GCP | Snowflake | Databricks |
|---|---|---|
| GCS bucket / prefix | Internal stage | Managed volume |
| GCS bucket + external table connection | External stage + storage integration | External volume + storage credential + external location |
| `gs://bucket/path` | `@db.schema.stage/path` | `/Volumes/catalog/schema/volume/path` |
| `gcloud storage cp ./f gs://…` | `PUT file://f @stage` | `databricks fs cp ./f dbfs:/Volumes/…` |
| `gcloud storage ls gs://…` | `LIST @stage` | `LIST '/Volumes/…'` |
| External table over the files | `SELECT $1 FROM @stage (FILE_FORMAT => …)` | `SELECT * FROM read_files('/Volumes/…', format => 'csv')` |
| `bq load` | `COPY INTO t FROM @stage` | `COPY INTO t FROM '/Volumes/…'` |
| GCS trigger → load | Snowpipe on a stage | Auto Loader on a volume |
| IAM on the bucket | `GRANT READ/WRITE ON STAGE` | `GRANT READ VOLUME / WRITE VOLUME ON VOLUME` |
| (none) | `FILE FORMAT` objects | Inline options (`sep`, `header`, `schemaHints`) |
| (none) | Table stage `@%t`, user stage `@~` | (none; only volumes you create) |

---

## 6. Transformation and dbt

| GCP | Snowflake | Databricks |
|---|---|---|
| Dataform | dbt Projects on Snowflake / dbt | dbt (native dbt task in Lakeflow Jobs) / Lakeflow pipelines in SQL |
| `dbt-bigquery` | `dbt-snowflake` | `dbt-databricks` |
| Scheduled queries | Tasks (scheduled SQL) | SQL task in a job |
| BigQuery scripting | Snowflake Scripting / stored procedures (SQL, Python, JavaScript) | SQL scripting / Python tasks |
| Dataform assertions | dbt tests, DMFs | dbt tests, contracts, unit tests, expectations |

### dbt adapter config

| dbt-bigquery | dbt-snowflake | dbt-databricks |
|---|---|---|
| `project` / `database` | `database` | `catalog` (alias `database`) |
| `dataset` / `schema` | `schema` | `schema` |
| Auth: OAuth / service-account keyfile | Auth: key pair / OAuth / password + MFA | Auth: OAuth (`auth_type: oauth`) / token, plus `host` and `http_path` |
| (none) | `warehouse` (per model: `snowflake_warehouse`) | `http_path` (per model: `databricks_compute`) |
| `partition_by={...}` | (none; automatic micro-partitions) | `partition_by=[...]` (discouraged when small) |
| `cluster_by=[...]` | `cluster_by=[...]` | `liquid_clustered_by=[...]` |
| Incremental: `merge`, `insert_overwrite`, `microbatch` | Incremental: `merge`, `append`, `delete+insert`, `insert_overwrite`, `microbatch` | Incremental: `merge`, `append`, `insert_overwrite`, `replace_where`, `microbatch` |
| `require_partition_filter`, `partition_expiration_days` | `transient`, `copy_grants` | `tblproperties` |
| `labels` | `query_tag`, object tags | `databricks_tags`, query tags |
| `grant_access_to` (authorized views) | `secure` views, `grants` | `grants` |
| `materialized_view` | `dynamic_table`, `materialized_view` | `materialized_view`, `streaming_table` |
| Python models via Dataproc | Python models via Snowpark | Python models, native |
| `persist_docs` → descriptions | `persist_docs` → comments | `persist_docs` → UC comments |

---

## 7. SQL dialect

| GoogleSQL (BigQuery) | Snowflake SQL | Databricks SQL |
|---|---|---|
| `` `project.dataset.table` `` | `database.schema.table` | `catalog.schema.table` |
| `INT64`, `FLOAT64`, `NUMERIC`, `STRING`, `BYTES`, `BOOL` | `NUMBER(38,0)`, `FLOAT`, `NUMBER(p,s)`, `VARCHAR`, `BINARY`, `BOOLEAN` | `BIGINT`, `DOUBLE`, `DECIMAL(p,s)`, `STRING`, `BINARY`, `BOOLEAN` |
| `JSON`, `JSON_VALUE(j, '$.a')` | `VARIANT`, `j:a::string` | `VARIANT`, `j:a::string` |
| `STRUCT` / `ARRAY` | `OBJECT` / `ARRAY` | `STRUCT` / `MAP` / `ARRAY` |
| `SAFE_CAST(x AS INT64)` | `TRY_CAST(x AS NUMBER)` | `try_cast(x AS BIGINT)` |
| `SAFE_DIVIDE(a, b)` | `DIV0(a, b)` / `DIV0NULL` | `try_divide(a, b)` |
| `IF(c, a, b)` | `IFF(c, a, b)` | `if(c, a, b)` / `iff(c, a, b)` |
| `IFNULL(a, b)` | `IFNULL` / `NVL` | `ifnull` / `nvl` / `coalesce` |
| `DATE_DIFF(end, start, DAY)` | `DATEDIFF(day, start, end)` | `datediff(DAY, start, end)` / `datediff(end, start)` |
| `DATE_ADD(d, INTERVAL 1 DAY)` | `DATEADD(day, 1, d)` | `date_add(d, 1)` / `d + INTERVAL 1 DAY` |
| `DATE_TRUNC(d, MONTH)` | `DATE_TRUNC('month', d)` | `date_trunc('MONTH', d)` |
| `TIMESTAMP_DIFF(a, b, HOUR)` | `TIMESTAMPDIFF(hour, b, a)` | `timestampdiff(HOUR, b, a)` |
| `FORMAT_DATE('%Y-%m', d)` | `TO_CHAR(d, 'YYYY-MM')` | `date_format(d, 'yyyy-MM')` |
| `PARSE_DATE('%Y%m%d', s)` | `TO_DATE(s, 'YYYYMMDD')` | `to_date(s, 'yyyyMMdd')` |
| `GENERATE_DATE_ARRAY(a, b)` | `GENERATOR` + `DATEADD` | `sequence(a, b, INTERVAL 1 DAY)` |
| `UNNEST(arr)` | `LATERAL FLATTEN(input => arr)` | `explode(arr)` / `LATERAL VIEW explode` |
| `ARRAY_AGG(x)` | `ARRAY_AGG(x)` | `array_agg(x)` / `collect_list(x)` |
| `STRING_AGG(x, ',')` | `LISTAGG(x, ',')` | `string_agg` / `listagg` (newer runtimes), `array_join(collect_list(x), ',')` |
| `REGEXP_CONTAINS(s, r)` | `REGEXP_LIKE(s, r)` / `RLIKE` | `regexp_like(s, r)` / `RLIKE` |
| `SELECT * EXCEPT (a)` | `SELECT * EXCLUDE (a)` | `SELECT * EXCEPT (a)` |
| `QUALIFY` | `QUALIFY` | `QUALIFY` |
| `MERGE` | `MERGE` | `MERGE INTO` (also `WHEN NOT MATCHED BY SOURCE`) |
| `FARM_FINGERPRINT(s)` | `HASH(s)` | `xxhash64(s)` / `hash(s)` |
| `GENERATE_UUID()` | `UUID_STRING()` | `uuid()` |
| (no identity column) | `AUTOINCREMENT` / sequences | `GENERATED ALWAYS AS IDENTITY` |
| `ML.GENERATE_TEXT` | `SNOWFLAKE.CORTEX.COMPLETE` / `AI_COMPLETE` | `ai_query()` |
| (none) | `AI_CLASSIFY`, `SUMMARIZE` | `ai_classify`, `ai_summarize`, `ai_extract` |

**Watch out**
- **`DATE_TRUNC` argument order:** BigQuery puts the date first. Snowflake and Databricks put the unit first.
- **Date format patterns** differ on all three platforms:
  - BigQuery: strftime (`%Y-%m-%d`)
  - Snowflake: `YYYY-MM-DD`
  - Databricks: Java (`yyyy-MM-dd`)
- **Excluding columns:** Snowflake uses `EXCLUDE`. BigQuery and Databricks use `EXCEPT`.
- **Hash values** differ across platforms. Never compare hashes between systems.

---

## 8. Orchestration and deployment

| GCP | Snowflake | Databricks |
|---|---|---|
| Cloud Composer / Airflow DAG | Task graph (root task + child tasks) | Lakeflow Job (DAG of tasks) |
| Cloud Scheduler | Task `SCHEDULE` (cron) | Job schedule (cron) |
| GCS / Eventarc trigger | Snowpipe auto-ingest, stream-triggered tasks | File-arrival trigger, table-update trigger |
| Airflow sensor | `WHEN SYSTEM$STREAM_HAS_DATA(...)` | Table-update trigger, condition task |
| Airflow clear and re-run | `EXECUTE TASK`, retry | **Repair run** |
| XCom | `SYSTEM$SET_RETURN_VALUE` | Task values, job parameters |
| Cloud Build + Terraform | Snowflake CLI (`snow`) + Terraform / schemachange | **Asset Bundles** (`databricks bundle deploy`) + Terraform |
| A project per environment | A database (or account) per environment | A catalog per environment + bundle targets |

---

## 9. Governance and security

| GCP | Snowflake | Databricks |
|---|---|---|
| Cloud IAM roles | RBAC roles (role hierarchy) | Unity Catalog privileges granted to groups |
| `bigquery.dataViewer` | `USAGE` on database + schema, `SELECT` | `USE CATALOG` + `USE SCHEMA` + `SELECT` |
| `bigquery.dataEditor` | `INSERT` / `UPDATE` / `CREATE TABLE` | `MODIFY` / `CREATE TABLE` |
| `bigquery.jobUser` | `USAGE` on warehouse | `CAN USE` on SQL warehouse |
| Dataset owner | `OWNERSHIP` (by role) | Owner (user, group or service principal) |
| Data Catalog / Dataplex | Snowsight catalog, Horizon | Unity Catalog / Catalog Explorer |
| Dataplex lineage | Access history + lineage (Horizon) | Built-in column-level lineage |
| Policy tags / data masking | **Masking policies** (incl. tag-based) | **Column masks** (UC functions), ABAC policies |
| Row access policies | **Row access policies** | **Row filters** (UC functions) |
| Resource labels | Object tags | UC tags |
| Cloud Audit Logs | `ACCOUNT_USAGE.ACCESS_HISTORY`, `LOGIN_HISTORY` | `system.access.audit` |
| VPC Service Controls | Network policies / network rules | Network policies, IP access lists |
| Private Service Connect | AWS / Azure / GCP PrivateLink | AWS PrivateLink |
| CMEK (Cloud KMS) | Tri-Secret Secure | Customer-managed keys (KMS) |
| Assured Workloads (HIPAA) | Business Critical edition | Compliance security profile + enhanced security |

---

## 10. Identity and secrets

| GCP | Snowflake | Databricks |
|---|---|---|
| Google Workspace users and groups | Users and roles (SCIM from your IdP) | Account users and groups (SCIM) |
| Service account | Service user (`TYPE = SERVICE`) | Service principal (plus an IAM role on AWS) |
| Service account key (JSON) | Key-pair authentication | OAuth M2M secret (avoid PATs) |
| Workload Identity Federation | Workload identity federation / external OAuth | Workload identity federation (GitHub OIDC → service principal) |
| Service account impersonation | `EXECUTE AS` owner (procedures), secondary roles | `run_as` on jobs and pipelines |
| A service account's access to GCS | **Storage integration** | **Storage credential** + external location |
| Secret Manager | `SECRET` objects | Secret scopes |
| `gcloud auth login` | `snow connection add` / SSO | `databricks auth login` |

---

## 11. Observability and cost

| GCP | Snowflake | Databricks |
|---|---|---|
| `INFORMATION_SCHEMA.JOBS` | `ACCOUNT_USAGE.QUERY_HISTORY` | `system.query.history` |
| Query execution details | Query Profile | Query profile |
| Billing export | `WAREHOUSE_METERING_HISTORY`, `METERING_DAILY_HISTORY` | `system.billing.usage` + `system.billing.list_prices` |
| Budgets and alerts | **Resource monitors** (can suspend warehouses), budgets | Account budgets, serverless budget policies |
| Cloud Monitoring alert policy | `CREATE ALERT` + notification integration | SQL Alerts, job and pipeline notifications, job health rules |
| Cloud Logging | Event tables | Job run output, pipeline event log (to a UC table) |
| Composer / Airflow UI | Task history (Snowsight) | Jobs & Pipelines → Runs |
| Dataplex auto data quality | Data metric functions | Data quality monitoring, expectations |

**Watch out**
- Snowflake resource monitors can **hard-stop** a warehouse.
- Databricks budgets **alert** but don't stop compute. In this lab, the `/cost-check` tripwires are the hard stop.

---

## 12. BI, AI and ML

| GCP | Snowflake | Databricks |
|---|---|---|
| Looker Studio / Looker | Snowsight dashboards, Streamlit | AI/BI Dashboards |
| LookML semantic layer | Semantic views | Metric views |
| Gemini in BigQuery / conversational analytics | Cortex Analyst / Snowflake Intelligence | Genie |
| Vertex AI endpoints / Model Garden | Cortex LLM functions | Model Serving, foundation model APIs, AI Gateway |
| Vertex AI Experiments | Snowflake ML experiments | MLflow Experiments |
| Vertex AI Feature Store | Snowflake Feature Store | Feature tables in UC |
| Vertex AI Model Registry | Snowflake Model Registry | Models in UC |
| Vertex AI Agent Builder | Cortex Agents | Agent Framework, Playground |

---

## 13. CLI and infrastructure as code

| GCP | Snowflake | Databricks |
|---|---|---|
| `gcloud auth login` | `snow connection add` | `databricks auth login --host <url> --profile <name>` |
| `gcloud config configurations` | `~/.snowflake/config.toml` connections | `~/.databrickscfg` profiles (`-p <name>`) |
| `bq ls` | `snow object list database` / `SHOW DATABASES` | `databricks catalogs list` / `schemas list` / `tables list` |
| `bq show` | `DESCRIBE TABLE` | `databricks tables get` |
| `bq mk --dataset` | `CREATE SCHEMA` | `databricks schemas create` |
| `bq query` | `snow sql -q` / SnowSQL | SQL Editor / statement execution API (`databricks api post /api/2.0/sql/statements`) |
| `gcloud storage cp` | `PUT` / `GET` (stage), `snow stage copy` | `databricks fs cp dbfs:/Volumes/…`, `aws s3 cp` |
| `gcloud run jobs execute` | `EXECUTE TASK` | `databricks bundle run` / `jobs run-now` |
| `gcloud projects get-iam-policy` | `SHOW GRANTS` | `databricks grants get` |
| Terraform `google` provider | Terraform `snowflakedb/snowflake` provider | Terraform `databricks` provider + `aws` provider |
| `google_bigquery_dataset` | `snowflake_schema` | `databricks_schema` |
| `google_storage_bucket` | `snowflake_stage` + `snowflake_storage_integration` | `aws_s3_bucket` + `databricks_external_location` / `databricks_volume` |
| `google_service_account` | `snowflake_service_user` | `databricks_service_principal` |
| `google_*_iam_member` | `snowflake_grant_privileges_to_account_role` | `databricks_grants` / `databricks_permissions` |

---

## 14. This lab's layers on each platform

| GCP | Snowflake | Databricks (this lab) |
|---|---|---|
| GCS bucket per source | External stage per source | `landing` volume per source (S3) |
| Raw dataset per source, loaded by Dataflow | Raw schema per source, loaded by Snowpipe | `<source_id>_bronze`, fed by one Lakeflow pipeline per source with expectations and quarantine |
| Staging and marts datasets built by dbt | Staging and marts schemas built by dbt | `silver` (sources unioned) and `gold` (marts) built by dbt |
| Monitoring dataset | Monitoring schema + event table | `observation` |
| Export dataset + GCS bucket | Outbound schema + unload to stage | `outbound` schema + outbound volume |
| A project per environment | A database per environment | `lab_dev` / `lab_preprod` / `lab_prod` catalogs |
| Composer DAG | Task graph | Lakeflow Job |
| Cloud Monitoring alerts | `CREATE ALERT` | Pipeline and job notifications, SQL Alerts |
| Terraform `google` | Terraform `snowflake` | Terraform `databricks` + `aws` |
