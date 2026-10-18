---
name: teardown
description: Tear the lab down safely and completely — bundle destroy, Terraform destroy in reverse stack order, then verify no AWS resources with the lab tag remain in us-west-2, no Databricks compute is left, and the trial is not converting to paid.
argument-hint: "[--keep-bootstrap]"
---

# /teardown

All destroy commands go to the user. Claude verifies between steps.

1. **Snapshot what's worth keeping:** a final `/cost-check` for `docs/evaluation.md`, exported dashboard JSON committed
   under `bundles/ops/`, and any query text saved in the repo. Confirm everything is committed and pushed.
2. **Bundles first,** so jobs and pipelines stop. For `ops`, `transform` and `ingestion`, in that order, and for each target in the order `prod`, `preprod`, `dev`:
   `databricks bundle destroy -t <target> -p lab --auto-approve`. Then verify with `databricks jobs list -p lab`
   and `databricks pipelines list-pipelines -p lab`: no lab resources should be left.
3. **Terraform in reverse,** with a plan reviewed before each one:
   `40-workspace-config` → `30-unity-catalog` → `20-account` → `10-workspace`. Then `00-bootstrap` last, unless
   `--keep-bootstrap` is set.
   - Stack 30: non-empty S3 buckets block deletion. Empty them first (`aws s3 rm --recursive`, which the user runs)
     or set `force_destroy` in a reviewed change.
   - Stack 00 holds the state bucket. Destroy it only once every other stack is gone. It needs `force_destroy` or an
     emptied, versioned bucket (delete the object versions too).
4. **Verify AWS:**
   `aws resourcegroupstaggingapi get-resources --tag-filters Key=project,Values=databricks-learning --region us-west-2`
   must return nothing. Also check `aws s3 ls` for leftover lab buckets, and look for `databricks`-named IAM roles.
5. **Verify Databricks:** in the account console, the workspace is gone (or cancelled) and there's no running compute.
   Under Subscription, the trial is **not** set to convert to paid. Ask the user to confirm this in the UI.
6. Log the final teardown and total spend in `docs/progress.md`, and add the rebuild-from-zero time if measured.
