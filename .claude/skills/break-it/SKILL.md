---
name: break-it
description: Run one pipeline breakage drill (new_column, missing_column, wrong_delimiter, bad_rows, empty_file, duplicate_file, volume_spike, late_file, corrupt_gzip, backfill_out_of_order) and verify that exactly the expected outcome happened — pipeline/job state, event-log expectation counts, quarantine rows, and the alert actually firing.
argument-hint: "<drill> [source_id] | list"
---

# /break-it

Prove the streaming layer fails loudly and quarantines quietly. A drill passes **only if the expected outcome
happened and nothing else did**. An alert that should have fired but didn't is a failure, even if the data
looks fine.

## Steps

1. **Load the drill.** Read `tests/drills/expected_outcomes.yml`. With `list` or no argument, show every drill,
   its last result from `docs/progress.md`, and stop.
   If the drill or its fixture doesn't exist yet, coach the user to write it. Don't write it for them.
   Each entry needs:
   - fixture path
   - target `source_id` and feed
   - expected pipeline update state, and expected flow state
   - expected counts per rule (passed, dropped, quarantined)
   - expected job and check-task state
   - which alert should fire, and through which channel
2. **Budget.** Run `/cost-check`. Drills should target the **dev** bundle and `lab_dev` unless the drill is
   explicitly about prod alerting.
3. **Baseline (read-only).**
   - The latest pipeline update id and state.
   - Current bronze and quarantine counts for the feed.
   - The last `observation` rows for the feed.
   - The time, which is the cutoff for "fired after".
4. **Show the prediction.** Print what should happen, step by step, before anything is dropped. Ask the user
   to write down what they *think* will happen if it differs. That's the learning moment.
5. **Drop.** Hand over the `aws s3 cp` (or `databricks fs cp`) command for the fixture. For `late_file`, the
   "drop" is skipping a cadence and waiting for the daily freshness alert.
6. **Observe.** Wait for the trigger, then verify each expected item.

   | Signal | How to check |
   |---|---|
   | Pipeline update and flow state | `databricks pipelines list-updates <id>`, or the event log `update_progress` / `flow_progress` |
   | Expectation counts per rule | `observation.expectations` for this update id, or raw `observation.pipeline_event_log` `details:flow_progress.data_quality.expectations` |
   | Quarantine rows and rule tags | `<source_id>_bronze.<feed>_quarantine` filtered by `_source_file` |
   | Rescued data, volume band, ledger, parity | `observation.rescued_data`, `observation.feed_volume`, `observation.file_ledger`, `observation.row_parity` |
   | Job and check-task state | `databricks jobs get-run <run_id>`, per task |
   | Alert fired | Ask the user to confirm the email or webhook arrived and paste the subject and timestamp. For SQL Alerts, `databricks alerts get <id>` shows the state. |

7. **Verdict.** Show a table of expected vs observed per signal, then PASS, or FAIL with the mismatching rows.
   Check for unexpected side effects too: other feeds affected, a double count, a stuck stream.
8. **Recover.** Coach the cleanup: remove or replace the fixture, `full_refresh` only the affected table if needed
   (cost-check first), confirm the stream is healthy again, and confirm counts are back to baseline plus the
   intended rows.
9. **Log** to `docs/progress.md`: the drill, source, expected vs observed, the verdict, and what changed (a new rule,
   a threshold, an alert). Add Databricks behaviour worth remembering to `docs/evaluation.md`. One example:
   how `addNewColumns` restarts a stream.

## Drill reference (the defaults; `expected_outcomes.yml` overrides them)

| Drill | Expected |
|---|---|
| `new_column` | With `addNewColumns`, the stream restarts with the new column and the change is noted. With `rescue`, the column lands in `_rescued_data`, the check task fails, and an email arrives. |
| `missing_column` | `expect_or_fail` trips → the update fails → a pipeline failure email arrives |
| `wrong_delimiter` | The single-column read trips `expect_or_fail` → fails loudly |
| `bad_rows` | Rows are quarantined with rule names. Under the threshold the job passes; over it the check task fails and an email arrives. |
| `empty_file` | The volume check flags `empty_drop` → the check task fails → an email arrives |
| `duplicate_file` | The ledger matches the hash → flagged. Parity shows no double count downstream. |
| `volume_spike` | The volume band is breached → the check task fails → an email arrives |
| `late_file` | The daily freshness Alert fires on its next run |
| `corrupt_gzip` | The flow fails → an `on-flow-failure` email arrives. Record whether the other flows in the update continued. |
| `backfill_out_of_order` | Ingested, and the ledger flags it as out of order. Incremental dbt spans are still correct after `/dbt-dev-loop`. |
