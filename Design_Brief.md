# Design Brief

## Scenario
We chose the Cross-Border Health Analytics scenario. The purpose of our application is to collect user inputted health symptoms and alert users when there is a health concern.

## Data Source/Volume
The primary data source is user inputted health symptoms, which is stored in JSON format. Volume TBA.

## Batch Schedule
Data will be ingested at 00:00 local time every day, due by 06:00.

## Service Level Agreement (SLA)

**Batch Performance**
- p95 completion time: **< 2.5 hours** per nightly run, including automatic retries.  
- End-to-end target: all 2 TB of uploads validated, transformed, and aggregated **before 06:00 local time** in each region.  
- Step Functions monitors runtime and triggers an alert if any job exceeds **3 hours**.

**Cost Boundaries**
- **< $16 per daily run** (≈ $440/month projected) — maintaining headroom under the **$500 ceiling**.  
- CloudWatch Budgets monitor cumulative spend and trigger SNS notifications if forecast > $480.

**Compliance & Retention**
- **Deletion jobs** triggered by Lambda must complete within **7 days (p95)** of user request, verified via CloudWatch metrics.  
- **Data residency:** no cross-region replication; each user’s data stays within its jurisdiction (`ca-central-1`, `eu-central-1`, or `sa-east-1`).  

**Reliability & Retry Policy**
- **Automatic retry limit:** 3 attempts for transient Lambda or Glue failures.  
- **Fail-fast rule:** jobs exceeding 3 retries abort and raise an alert to operations.  
- SLA breaches (runtime, cost, or retention) are logged to the **Ethics Debt Ledger** and reviewed weekly.
