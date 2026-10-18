---
name: bundle-ship
description: Ship an Asset Bundle (ingestion, transform, or ops) through dev → preprod (main) → prod (release). Dev is deployed by hand; preprod and prod are promoted through git (merge to main, PR main → release) and deployed by GitHub Actions. Reviews the resolved config, then verifies bundle summary, run state, and output tables.
argument-hint: "<ingestion|transform|ops> <dev|preprod|prod> [resource_key]"
---

# /bundle-ship

Asset Bundles are also called Declarative Automation Bundles in newer docs. Work from `bundles/<bundle>/`.
The environment model is in CLAUDE.md under "Environments". **Only `dev` is ever deployed from a laptop.**

## Every target: validate and review (Claude runs)

1. For preprod and prod, run `/cost-check` first.
2. `databricks bundle validate -t <target> -p lab -o json`. Read the resolved config and check:
   - **Catalog per target:** dev → `lab_dev`, preprod → `lab_preprod`, prod → `lab_prod`. Nothing in dev or preprod
     writes to `lab_prod`; they may only read the `lab_prod.*_bronze` schemas.
   - **Pipelines:** one per source, each publishing only to `<catalog>.<source_id>_bronze`. `serverless: true`, `continuous: false`, the event log published to `<catalog>.observation.pipeline_event_log`,
     and failure notifications set. In dev and preprod, the pipeline points at its own catalog's golden-drop `landing`
     volume, not the prod landing.
   - **Jobs:** serverless environments only, with `email_notifications.on_failure`.
     - **File-arrival triggers and schedules** are paused in dev (`mode: development`) and explicitly paused in preprod
       (`pause_status: PAUSED`). They are live only in prod.
   - **dbt tasks:**
     - `--target` matches the bundle target.
     - Selectors are scoped in dev.
     - preprod does a full `dbt build`.
     - `dbt-databricks` is pinned.
   - **Identities:**
     - `dev`: `mode: development`.
     - `preprod`/`prod`: `mode: production`.
     - `prod`: `run_as` is the `prod-runner` SP.
   - **Secrets:** no hard-coded hosts or tokens.

## dev (you deploy)

3. You run `databricks bundle deploy -t dev -p lab`. Then Claude runs `bundle summary -t dev -p lab`.
4. Optionally, you run `databricks bundle run -t dev -p lab <resource_key>`. Use a full refresh only when it's needed.
5. Verify. Continue at the shared verification step.

## preprod (merge to `main`)

3. Check the feature branch's PR checks are green: validate, unit tests, and slim CI into `lab_dev.ci_pr<N>_*`
   (`gh pr checks <N>`). Review the diff with the user.
4. The user merges the PR into `main`. Watch the deploy workflow with `gh run list --branch main` and `gh run view <id>`.
   It deploys `-t preprod`, runs a full `dbt build` in `lab_preprod`, and runs the golden-drop integration test.
5. Verify. Continue at the shared verification step, against `lab_preprod`.

## prod (PR `main` → `release`)

3. Preprod must be green for the exact `main` commit being promoted. Compare `lab_preprod` and `lab_prod` row
   counts and key aggregates for the changed models, and explain any differences.
4. The user opens the promotion PR: `gh pr create --base release --head main --title "release: <summary>"`.
   The body includes the preprod run link and the diff summary. It's approved and merged; the `prod` GitHub
   environment gate applies.
5. Watch the prod deploy workflow (`gh run list --branch release`). It deploys `-t prod` as `prod-deployer`.
   Then Claude runs `bundle summary -t prod -p lab`.
6. Verify. Continue at the shared verification step, against `lab_prod`. Confirm the triggers are live.
7. **Hotfix path,** if needed: branch from `release`, PR into `release`, then PR `release` → `main` so the two
   don't drift.

## Shared verification (Claude runs)

- `databricks jobs get-run <run_id>` or `pipelines list-updates <id>`. Every task and flow is green.
- Output tables in the target catalog have a fresh `_ingested_at` / `max(updated_at)` and plausible counts.
- `<catalog>.observation.expectations` for the run shows no unexpected `expect_or_fail` trips.
- If something failed, switch to `/triage-run`. Otherwise log the target, commit SHA and result to `docs/progress.md`.
