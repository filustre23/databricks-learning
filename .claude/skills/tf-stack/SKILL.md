---
name: tf-stack
description: Terraform runbook for one stack under terraform/ — fmt, validate, plan, then review the plan against the lab's hard rules (us-west-2 only, tagged, serverless, standalone, no surprise destroys) before handing the user the apply command and confirming zero drift afterward.
argument-hint: "<00|10|20|30|40|stack-dir>"
---

# /tf-stack

Resolve `$ARGUMENTS` to `terraform/<NN-*>/`. Claude runs the read-only steps. `apply`, `destroy`, `import` and
`state` edits go to the user.

1. **Static checks (Claude runs):**
   - `terraform -chdir=<stack> fmt -check -recursive`
   - `terraform -chdir=<stack> validate`

   If `init` is needed, hand it over (it writes `.terraform/`).
2. **Review the HCL** the user wrote before planning:
   - Every AWS provider pins `region = "us-west-2"` and sets `default_tags` (project, owner, ttl).
   - No `data` sources or variables point at pre-existing infrastructure. Everything is created here or read
     from this repo's own remote state.
   - No secrets or account IDs are hard-coded; they come from gitignored `*.tfvars` or env vars.
   - Databricks: the workspace uses `compute_mode = "SERVERLESS"`. There are no `databricks_cluster`
     resources and no `databricks_model_serving`. The SQL warehouse is serverless, 2X-Small, with auto-stop ≤ 5 minutes.
   - UC: workspace-level resources `depends_on` the metastore assignment. Storage credential IAM roles use the
     self-assuming trust pattern.
3. **Plan (Claude runs):** `terraform -chdir=<stack> plan -out=tfplan -no-color`. Then
   `terraform -chdir=<stack> show -json tfplan` and read it.
4. **Plan review.** Summarize creates, updates and destroys. Flag loudly:
   - any **destroy** or **replace**
   - any resource outside us-west-2
   - any AWS resource missing the project tag
   - classic compute, NAT gateways or EC2
   - IAM `*` actions or resources wider than the lab buckets
   - S3 buckets without public access block or versioning (state bucket)
   - anything that looks like it touches non-lab resources

   Explain *why* each flagged item matters, so the user learns from it.
5. **Hand over apply:** `terraform -chdir=<stack> apply tfplan`. The user runs it and pastes the tail of the output.
6. **Drift check (Claude runs):** `terraform -chdir=<stack> plan -detailed-exitcode`. Exit code 0 is required.
   Exit code 2 means drift: explain it.
7. **Verify** with the matching CLI. For example, stack 30: `databricks catalogs list -p lab`,
   `databricks volumes list lab_prod landing -p lab`, `aws s3 ls --region us-west-2`.
8. Remind the user to commit the HCL and the lock file, and never `*.tfvars` or state. Log it in `docs/progress.md`.
