# Learning log

A running record of the Databricks learning path: which concepts were covered, which questions came up, what
clicked, and what that means for the `databricks-intro` module in
[knowledge-builder](https://github.com/filustre23/knowledge-builder). `docs/progress.md` tracks the *build*
(phases and checks). This file tracks the *learning*.

## How to read an entry

| Field | Meaning |
|---|---|
| **Covered** | Concepts taught in the step |
| **Questions asked** | The learner's own questions, close to verbatim. These are the best signal of where a lesson needs to start. |
| **What clicked** | The explanation or analogy that landed |
| **Module impact** | Proposed change to a knowledge-builder lesson (order, framing, a new lesson) |

---

## Session 1: orientation and ingestion concepts

**Covered**
- Unity Catalog hierarchy: metastore → catalog → schema → table / volume
- Databricks has no storage of its own. Tables and volumes are files in the cloud object store (S3 on AWS).
- Managed vs external volumes; why the production landing zone is the company's own S3 bucket
- Environments as catalogs (`lab_dev` / `lab_preprod` / `lab_prod`)
- Files land once, in prod. dev and preprod read prod bronze and test ingestion with golden drops.
- Medallion naming: per-source bronze schemas → silver (union) → gold; `observation` and `outbound`
- The ingestion toolkit: volume → `read_files()` → `COPY INTO` → Auto Loader → Lakeflow pipeline

**Questions asked**
- "What are catalogs?"
- "Why S3?" (asked about the ingestion diagram)
- "What is a volume?"
- "What is lab_prod? What does it mean?"
- "The files load in prod because it's not like we can tell a client to land it in three different buckets"
  (the learner reasoned this out before it was explained)
- "What is ops?" → the schema was renamed `observation`

**What clicked**
- **Volume = Snowflake stage.** The learner made this connection themselves and it landed immediately.
- **Catalog = BigQuery project / Snowflake database.** Mapping to platforms they already know worked better than
  defining the term in the abstract.
- **"Clients land files in one place; environments are internal."** This framed the whole environment model.

**Module impact (proposed for `databricks-intro`)**
1. **Add "Where data actually lives" before lesson 02.** The S3 question came before the catalog question. Start with
   "Databricks is compute + governance over your object store", then Unity Catalog.
2. **Lead every concept with the platform analogy** (BigQuery / Snowflake column first), then the Databricks specifics.
   Link `docs/platform-equivalents.md` from each lesson.
3. **Teach environments early (after lesson 02), not in lesson 13.** Names like `lab_prod` confused the learner until
   environments were explained. Teach the land-once-in-prod principle with them.
4. **Lesson 03 (Volumes):** lead with the stage analogy, plus the managed vs external table.
5. **Explain naming conventions explicitly** (`<source>_bronze`, `observation`, `outbound`) when they first appear.
   Unexplained names generated questions.

**Questions asked (hands-on)**
- "These files would normally be in an S3 bucket, correct?" (after generating the Source B files)
- "So Databricks doesn't actually have a warehouse? Everything is in S3 and we transform within it as if it was the warehouse?"
  - The first answer (a four-layer stack diagram + tables) was "kinda hard to understand". The re-explanation that worked, confirmed by the learner ("easy to understand this way"): a Snowflake comparison table + "a table = a folder of files + a logbook".

**Module impact (added)**
6. **In lesson 03 (Volumes), show the production path next to the sandbox path:**
   `s3://<landing>/<source>/<feed>/yyyy/mm/dd/` ↔ `/Volumes/<env>/landing/<source>/…`. Upload the sandbox files
   into feed/date folders from the start, so the learner practices the real layout.

7. **Make the new "Where data actually lives" lesson cover the full lakehouse stack:** storage (S3) → table format
   (Delta: Parquet + `_delta_log`) → compute (SQL warehouse = compute only, like a Snowflake virtual warehouse) →
   governance (Unity Catalog). The learner's mental model was "no warehouse, just S3". The missing pieces were Delta
   and the fact that a "SQL warehouse" is only compute. Exercise: `DESCRIBE DETAIL` / `DESCRIBE HISTORY` on the
   first loaded table.
8. **Teaching style: simple first, detail on request.** Lead with one sentence, one comparison to a platform the
   learner knows, and one tiny picture. Save the layered diagrams and feature tables for "going further". Dense
   multi-table answers lost the learner; "table = folder + logbook" landed.

---

## Session 1 (continued): hands-on

| Step | Status | Notes |
|---|---|---|
| 0. CLI profile `lab` (`databricks auth login`) | ✅ Done | Profile valid; user active |
| 1. Generate Source B (Aetna layout) locally | ✅ Done | 3 files, 68,909 rows, 106 seeded defects. Eligibility is member-month grain (44k rows ≈ 2k members × 24 months); the medical file is 178 columns wide. |
| 2. Sandbox schema + managed volume + `fs cp` | ⏳ Next | |
| 3. `read_files()` profiling | | |
| 4. `COPY INTO` | | |
| 5. Auto Loader | | |
| 6. Lakeflow pipeline + expectations | | |
