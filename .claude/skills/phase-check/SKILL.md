---
name: phase-check
description: Verify a plan phase's "done when" criteria with read-only commands, record pass/fail and notes in docs/progress.md, and name the next step. Use at the end of each phase or to see where the lab stands.
argument-hint: "<phase 0-9 | 4b> | status"
---

# /phase-check

1. Read the phase's **Done when** criteria from `docs/plan.md`. If the user hasn't committed the plan yet, ask them
   to paste it or commit it first.
2. With `status`, read `docs/progress.md` and report each phase's last result plus the next unchecked item. Then stop.
3. Turn each criterion into a concrete check. Prefer evidence Claude can gather read-only:
   - **Infra:** `terraform plan` shows no changes for each relevant stack, and the tag query returns only us-west-2 resources.
   - **Workspace and UC:** `databricks current-user me`, `catalogs list`, `schemas list`, `volumes list`, `grants get`.
   - **Files:** `aws s3 ls`, `databricks fs ls dbfs:/Volumes/...`.
   - **Pipelines and jobs:** `pipelines list-updates`, `jobs list-runs`, `jobs get-run`.
   - **Data:** SELECT counts through the Databricks SQL MCP or dbt `show`.
   - **dbt:** `dbt build`/`test` results (through `/dbt-dev-loop`), plus contract enforcement.
   - **Alerts:** ask the user to confirm the notification arrived (subject and timestamp).
   - **CI:** `gh run list` and `gh run view` on the repo.
4. Show a checklist table: criterion → evidence → PASS/FAIL. Don't mark PASS on the user's word alone when
   evidence can be gathered. For things only the user can see (an email, a UI state), record "user-confirmed".
5. Append to `docs/progress.md` (create the file if it doesn't exist): the date, phase, each criterion's result,
   the open issues, and the cost-to-date from the last `/cost-check`.
6. Name the next step: the first FAIL, or the first task of the next phase.
