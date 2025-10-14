# Design Brief

## Scenario
**Cross-Border Health Analytics**: The purpose of our data ingestion pipeline is to collect user-submitted health symptom data from Canada, the EU, and Brazil, to later transform into insights that help identify health risks for users.

## Data Source/Volume
The primary data source is user inputted health symptoms from a cross-region app. We anticipate a total daily ingest volume of 2 TB per night from Canada, the EU, and Brazil.

## Batch Schedule
Step functions orchestrate AWS Glue ETL jobs to ingest data at 00:00 every day, due by 06:00 (local time for both).

## Service Level Agreement (SLA)
**Batch Performance**
- p95 completion time: **< 5 hours** per nightly run, including automatic retries and AWS Glue cold start 
- End-to-end target: all 2 TB of uploads validated, transformed, and aggregated **before 06:00 local time** in each region.  
- Step Functions monitors runtime and triggers an alert if any job exceeds **3 hours**.

**Cost Boundaries**
- Total cost <$500, with a daily target **< $16 per run** (≈ $440/month projected)
- CloudWatch Budgets monitor cumulative spend and trigger SNS notifications if forecast > $480.

## Design Feasibility
- The required sustained throughput to process 2 TB (2,048 GB) within 4 hours is:
    - 2,048 GB * 1,024 MB/GB = 2,097,152 MB
    - After 5x data compression, the amount of data is 409.6 GB
    - 5 hours = 5 * 3,600 s = 18,000 s
    - Therefore, the required sustained throughput is 2,097,152 MB / 18,000 s = 23.30 MB/s
- AWS Glue exposes DPUs where 1 DPU = 4 vCPUs + 16 GB
    - Assuming 3 MB/s per vCPU, we need 7.8 vCPUs, which means we need to configure 2 DPUs
    - The cost of 2 DPUs is ~$4.40, amounting to a totl monthly (30 runs) cost of **$132** for compute only. This is under the required cost SLA.


**Compliance & Retention**
- **Deletion jobs** triggered by Lambda must complete within **7 days (p95)** of user request, verified via CloudWatch metrics and acceptance tests.
- **Data residency:** no cross-region replication; each user’s data stays within its jurisdiction (`ca-central-1`, `eu-central-1`, or `sa-east-1`).
- **Consent**: users must give explicit consent to provide metadata and telemetry.
    - Our app provides an opt-in button in the sign up page to explicitly ask users for consent.
    - Users are also given the option to opt-out at any time by navigating to the `settings` section of the app.

**Reliability & Retry Policy**
- **Automatic retry limit:** 3 attempts for transient Lambda or Glue failures.  
- **Fail-fast rule:** jobs exceeding 3 retries abort and raise an alert to operations.
- SLA breaches (runtime, cost, or retention) are logged to the **Ethics Debt Ledger** and reviewed weekly.

**Regulatory Deadlines**
- **Canada (PIPEDA):** Deletion or consent withdrawal requests are processed within 7 days, based on our internal SLA target that interprets PIPEDA’s requirement to act “promptly.” This ensures user requests are handled in a timely and measurable way, even though the regulation does not define an exact deadline.
- **EU (GDPR Art. 12–17):** Data-subject requests (access, deletion, correction) completed within **30 days**; monitored through CloudWatch deletion metrics.  
- **Brazil (LGPD Art. 18):** Data-subject deletion requests honored within **7 days**, logged to `DeletionRequestLog` for audit.  
- All regions: nightly aggregation jobs must complete **before 06:00 local time** to ensure regulators could audit fresh, up-to-date datasets on request.
