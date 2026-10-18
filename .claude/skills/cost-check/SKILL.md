---
name: cost-check
description: How much of the $400 trial budget is spent? Totals Databricks usage from system.billing.usage against the cap, shows the trend by SKU and day, applies the $150/$250/$325 tripwires, and checks AWS spend for the lab tag. Run before any expensive step.
argument-hint: "[days]"
---

# /cost-check

1. **Databricks usage.** Run this through the Databricks SQL MCP, or hand the user the SQL to run in the SQL Editor.
   Running it wakes the warehouse; that's fine, but don't loop it.
   ```sql
   SELECT u.usage_date, u.sku_name, u.billing_origin_product,
          SUM(u.usage_quantity)                                   AS dbus,
          SUM(u.usage_quantity * lp.pricing.effective_list.default) AS list_usd
   FROM system.billing.usage u
   LEFT JOIN system.billing.list_prices lp
     ON u.sku_name = lp.sku_name AND u.usage_unit = lp.usage_unit
    AND u.usage_end_time >= lp.price_start_time
    AND (lp.price_end_time IS NULL OR u.usage_end_time < lp.price_end_time)
   WHERE u.usage_date >= current_date() - INTERVAL 14 DAYS
   GROUP BY ALL ORDER BY u.usage_date DESC, list_usd DESC
   ```
   If `effective_list` isn't populated, fall back to `pricing.default`. Billing tables lag by a few hours, so also
   ask the user for the figure from the account console (Usage page). Treat the higher of the two as the truth.
2. **AWS spend.** Hand over `aws ce get-cost-and-usage` filtered on tag `project=databricks-learning` for the trial
   window, or ask for the AWS Budgets figure. It should be under $5. Anything more means something non-serverless
   exists (NAT, EC2). Find it with
   `aws resourcegroupstaggingapi get-resources --tag-filters Key=project,Values=databricks-learning --region us-west-2`.
3. **Report.** Show:
   - total spent / $400
   - % used and days of trial left
   - the burn rate per day over the last 3 days
   - the projected total at trial end
   - the top 3 SKUs
   - the single most expensive object, if it can be identified (`usage_metadata.job_id`, `warehouse_id`, `dlt_pipeline_id`)
4. **Tripwires.**
   - **Under $150:** OK.
   - **$150–250:** caution. Use scoped dbt selects only, and no stretch work.
   - **$250–325:** warn. Finish the current phase only, and skip AI/ML and Genie.
   - **Over $325, or a projection over $400:** **STOP.** Say so plainly and recommend `/teardown`.
5. Append one line to the cost table in `docs/evaluation.md`: date, total, burn per day, top SKU.
