---
name: triage-run
description: Diagnose a failed or suspicious Lakeflow job run or pipeline update — pull run/update state and the event log, classify the failure (schema drift, rescued data, expectation, corrupt file, dbt test, permissions, budget), walk the fix checklist, and hand the user repair-run.
argument-hint: "<job name|job_id|run_id|pipeline name|pipeline_id>"
---

# /triage-run

1. **Locate (read-only).**
   - Job: `databricks jobs list-runs --job-id <id> -p lab`, then `jobs get-run <run_id>` and `jobs get-run-output <task_run_id>` for the failed task.
   - Pipeline: `databricks pipelines list-updates <id>`, then the events for the failed update from
     `observation.pipeline_event_log` (`level IN ('ERROR','WARN')`, `event_type IN ('update_progress','flow_progress')`,
     ordered by timestamp), or `databricks pipelines list-pipeline-events <id>`.
2. **Classify** by the first real error, not the last cascade:

   | Class | Signature | Usual fix |
   |---|---|---|
   | Schema drift | `UnknownFieldException` / stream restarted after new columns | Expected with `addNewColumns`: rerun. Otherwise update the registry `expected_columns`/mode. |
   | Rescued data | Check task failed on `rescued_data` | Inspect `_rescued_data`, then fix the registry schema or the parser |
   | Expectation fail | `expect_or_fail` violation in `flow_progress` | Find the file and rule. Is the file bad, or is the rule wrong? |
   | Quarantine threshold | Check task failed on the quarantine % | Examine the quarantine rows by rule; fix upstream, or adjust the threshold if it's justified |
   | Corrupt or empty file | Gzip/Parquet read error, `empty_drop` | Replace the file and confirm the ledger |
   | dbt test or contract | dbt task failed | Run `/dbt-dev-loop` on the failing model |
   | Permissions | `PERMISSION_DENIED`, `USE CATALOG` | Grant through Terraform (stack 30), not the UI |
   | Budget or compute | Serverless quota, budget policy | Run `/cost-check` |

3. **Explain** what happened and why, with the evidence lines.
4. **Fix checklist.** Coach the change; the user writes it. Re-verify with a dev deploy if code changed (`/bundle-ship`).
5. **Recover.**
   - For a job, hand over `databricks jobs repair-run <run_id> --rerun-tasks <task_key>` (or `--rerun-all-failed-tasks`).
   - For a pipeline, hand over `databricks pipelines start-update <id>`. Add `--full-refresh-selection <table>` only if the
     checkpoint or state is genuinely broken, and run `/cost-check` first.
6. **Confirm** green, check that row parity reconciles, and confirm the alert that should have fired actually did.
   If it didn't, that's a second bug: fix the notification config.
7. Log the incident to `docs/progress.md`: the symptom, root cause, fix and time to recover. If a new breakage class
   showed up, propose a matching drill for `tests/drills/`.
