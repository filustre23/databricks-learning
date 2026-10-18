---
name: dq-scorecard
description: Score the lab's data-quality coverage — compare dbt test failures and pipeline expectation/quarantine results against the Tuva synthetic generator's seeded-defect issue_manifest.csv, and report the catch rate per defect class and the gaps.
argument-hint: "[source_id]"
---

# /dq-scorecard

1. **Find the manifests.** Each Tuva-generated source has an `issue_manifest.csv` in its generator output (sources b and c).
   It should also be loaded as a dbt seed or table (`observation.seeded_defects`). If it isn't loaded yet, coach the user to
   add it as a seed.
2. **Gather the detections:**
   - Pipeline: quarantine rows by rule, and `expect` warnings, from `observation.expectations`.
   - dbt: failing rows from `store_failures` tables (`lab_dev`, `lab_preprod` or `lab_prod` `.dbt_test_failures.*`), keyed back to a
     claim or member id.
   - If it exists, the user's `observation.dq_scorecard` model. Prefer it, and check its logic.
3. **Match** each seeded defect (defect class + record key) to a detection on the same key. Report:
   - for each defect class: seeded count, detected count, catch rate, and which test or rule caught it
   - **missed classes**, with a proposed test or expectation for each (the user writes it)
   - **false positives:** detections on records with no seeded defect. Explain whether they're real issues the
     generator didn't list, or over-strict rules.
4. Write the headline (overall catch rate, classes covered) to `docs/evaluation.md`, and list the gaps in
   `docs/progress.md` as next tasks.
